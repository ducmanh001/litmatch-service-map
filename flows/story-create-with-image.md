# Đăng story kèm ảnh và dọn story hết hạn

id: flow:story-create-with-image
group: Xã hội & nội dung
services: module:feed, module:media

Story chỉ được lưu khi ảnh đã được Media xác nhận là của đúng chủ, đúng purpose Story và đã upload hợp lệ; sau đó một sweeper nền của Feed xoá cứng các story hết hạn nên Feed và Media phải phối hợp.

Client gửi mediaAssetId, Feed nhờ Media xác nhận và đổi thành URL ảnh rồi mới lưu Story; sweeper định kỳ xoá story hết hạn.

## Participants

- web (client): Web client
- feed (service): Feed
- media (service): Media
- postgres (infra): PostgreSQL

## Steps

1. web -> feed [call/http]: POST /stories kèm mediaAssetId, caption, audience và Idempotency-Key (Route yêu cầu JWT (JwtAuthGuard toàn cục). Guest không gọi Media và bị StoryService.assertNotGuest chặn.) {apps/core-api/src/app/app.module.ts:154}
2. feed -> media [call/di-call]: resolveImageUrl(userId, mediaAssetId, purpose Story) {apps/core-api/src/modules/feed/controllers/story.controller.ts:57}
3. media -> postgres [call/shared-db]: Tìm asset theo id và ownerUserId; sai chủ hoặc sai purpose thì 404 {apps/core-api/src/modules/media/media.service.ts:85}
4. media -> postgres [call/shared-db]: Kiểm tra file đã upload, chuyển asset sang trạng thái Ready (Khi asset chưa Ready và provider không phải dev, Media kiểm tra size/contentType và promote file rồi mới lưu Ready.) {apps/core-api/src/modules/media/media.service.ts:126}
5. media -> feed [return/di-call]: Trả URL công khai của ảnh đã xác nhận
6. feed -> postgres [call/shared-db]: Lưu Story với mediaUrl, expiresAt theo STORY_TTL_HOURS, idempotencyKey (Vi phạm unique idempotencyKey: trả story cũ nếu cùng mediaUrl, khác thì 409.) {apps/core-api/src/modules/feed/services/story.service.ts:61}
7. feed -> web [return/http]: Trả StoryDto vừa tạo
8. feed -> postgres [async/shared-db]: Sweeper theo chu kỳ: DELETE story hết hạn, tối đa 500 dòng mỗi tick (Chạy bằng interval từ STORY_SWEEPER_INTERVAL_MS; story_views tự cascade xoá theo FK.) {apps/core-api/src/modules/feed/jobs/story-sweeper.service.ts:60}

## Invariants

- Story chỉ nhận assetId; Media là nơi duy nhất xác nhận asset đúng chủ và đúng purpose Story, và (khi MEDIA_STORAGE_PROVIDER khác 'dev') đã upload hợp lệ với size/contentType khớp, trước khi Feed lưu mediaUrl; provider dev bỏ qua kiểm tra file.
- Feed gọi Media trước khi lưu Story, nên asset không hợp lệ sẽ bị từ chối mà không tạo bản ghi Story nào.

## Failure modes

- when Asset không tồn tại, sai chủ hoặc sai purpose: Media ném ASSET_NOT_FOUND (404), Feed không lưu story.
- when Ảnh chưa upload xong hoặc size/contentType không khớp: Media ném ASSET_NOT_READY (422), story không được tạo.
- when Cùng Idempotency-Key nhưng mediaUrl khác: Feed trả 409 POST_IDEMPOTENCY_CONFLICT.
- when Một tick sweeper lỗi: Lỗi được log qua logger.error, tick sau chạy lại; read-path vẫn lọc story hết hạn nên đúng đắn không bị ảnh hưởng.

## Endpoints covered

- POST /stories [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- StorySweeperService.onApplicationBootstrap [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
