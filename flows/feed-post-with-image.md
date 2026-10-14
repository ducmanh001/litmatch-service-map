# Đăng bài kèm ảnh: upload presigned rồi gắn asset

id: flow:feed-post-with-image
group: Xã hội & nội dung
services: module:media, module:feed, module:user

Client không gửi file qua API: xin presigned URL từ media, upload thẳng lên object storage, rồi đăng bài chỉ với assetId. Feed không tự tin asset nên phải hỏi media xác thực trước khi lưu imageUrl, và hỏi user để dựng thông tin tác giả cho phản hồi.

Ba pha: media cấp presigned URL, client upload thẳng lên storage, rồi feed nhờ media xác thực asset trước khi lưu bài.

## Participants

- web (client): Web client
- media (service): Media
- feed (service): Feed
- user (service): User
- external:object-storage (infra): Object storage
- postgres (infra): PostgreSQL

## Steps

1. web -> media [call/http]: POST /media/images/upload-intent xin presigned URL (Route REST đi qua JwtAuthGuard toàn cục.) {apps/core-api/src/app/app.module.ts:154}
2. media -> external:object-storage [call/sdk]: Cấp presigned PUT URL cho storageKey mới (Gọi qua ImageStoragePort.issueUploadUrl; provider s3 ký URL cục bộ bằng s3-request-presigner (không có round-trip mạng tới storage), việc này chạy TRƯỚC khi lưu asset. Guest, contentType và size không hợp lệ bị từ chối trước đó.) {apps/core-api/src/modules/media/media.service.ts:64}
3. media -> postgres [call/shared-db]: Lưu ImageAsset trạng thái Pending (owner, purpose, size) {apps/core-api/src/modules/media/media.service.ts:68}
4. media -> web [return/http]: Trả assetId, uploadUrl, expiresAt
5. web -> external:object-storage [call/http]: PUT ảnh thẳng lên uploadUrl, không qua core-api
6. web -> feed [call/http]: POST /feed/posts với content, imageAssetId và Idempotency-Key {apps/core-api/src/modules/feed/feed.controller.ts:66}
7. feed -> media [call/di-call]: Nếu có imageAssetId (và không phải guest): resolveImageUrl(purpose Post) (Controller chỉ gọi media khi user không phải guest và body có imageAssetId; không có ảnh thì bỏ qua cả pha media (các bước 7-11).) {apps/core-api/src/modules/feed/feed.controller.ts:66}
8. media -> postgres [call/shared-db]: Tìm asset theo id và ownerUserId (Không thấy hoặc sai purpose đều trả ASSET_NOT_FOUND.) {apps/core-api/src/modules/media/media.service.ts:85}
9. media -> external:object-storage [call/sdk]: Provider khác dev: head, kiểm size/MIME/magic bytes, promote sang key cuối (Chỉ khi asset chưa Ready và MEDIA_STORAGE_PROVIDER khác dev: head key cuối; chưa có thì assertUploadedImage (head + readPrefix) rồi promote. Sai thì xóa object và ném ASSET_NOT_READY. Provider dev bỏ qua toàn bộ kiểm tra này và chỉ đổi storageKey.) {apps/core-api/src/modules/media/media.service.ts:113, apps/core-api/src/modules/media/media.service.ts:114}
10. media -> postgres [call/shared-db]: Đánh dấu asset Ready với storageKey cuối (Với provider dev, asset được đánh dấu Ready mà không kiểm tra object đã upload; asset đã Ready thì bỏ qua cả bước kiểm và lưu.) {apps/core-api/src/modules/media/media.service.ts:126}
11. media -> feed [return/di-call]: Trả imageUrl công khai đã xác thực
12. feed -> postgres [call/shared-db]: Lưu Post với imageUrl và idempotencyKey (Trùng idempotencyKey (unique violation): trả lại bài cũ nếu content và imageUrl giống hệt, khác thì ném 409 POST_IDEMPOTENCY_CONFLICT. Post save không chung transaction với asset Ready ở media.) {apps/core-api/src/modules/feed/feed.service.ts:66}
13. feed -> user [call/di-call]: getByIdOrThrow để dựng PublicProfileDto tác giả {apps/core-api/src/modules/feed/feed.controller.ts:208}
14. feed -> web [return/http]: Trả PostDto kèm thông tin tác giả

## Invariants

- Feed chỉ lưu imageUrl do media.resolveImageUrl trả về, nên client không thể tự gửi URL tùy ý; media xác nhận asset đúng chủ và đúng purpose, còn kiểm tra đã upload/size/MIME/magic bytes chỉ chạy khi MEDIA_STORAGE_PROVIDER khác dev.
- Upload đi thẳng từ client lên object storage bằng presigned URL nên core-api không nhận body nhị phân; việc xác thực phải làm ở bước đăng bài, trước khi ghi Post.
- resolveImageUrl chạy ở controller trước FeedService.createPost, nên asset hỏng làm bài không được ghi vào DB.

## Failure modes

- when Asset không tồn tại, sai chủ hoặc sai purpose: media ném ASSET_NOT_FOUND (404), bài không được tạo
- when Ảnh chưa upload xong, size/contentType lệch hoặc magic bytes không khớp: media xóa object (nếu có) và ném ASSET_NOT_READY (422), bài không được tạo
- when Đăng bài lặp với cùng idempotencyKey: unique violation được bắt và feed đọc lại bài đã tạo thay vì tạo bản sao
- when Body không có content lẫn imageUrl: feed ném POST_CONTENT_REQUIRED (422)
- when Lưu Post lỗi sau khi media đã promote ảnh: Asset đã được lưu trạng thái Ready (và object đã promote) nhưng Post không được tạo; hai thao tác không chung transaction. Suy luận: asset mồ côi này không có job dọn trong module media.

## Endpoints covered

- POST /media/images/upload-intent [module:media] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/media.md
- POST /feed/posts [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
