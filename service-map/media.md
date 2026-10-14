# Service map: Media (uploads)

id: service-map:media
module: module:media
deployable: core-api
confidence: high

Presigned upload ảnh, xác thực asset

Cấp presigned PUT URL để client upload ảnh thẳng lên object storage và là nơi DUY NHẤT xác nhận asset hợp lệ (đúng chủ, đúng purpose, đã upload, MIME/size khớp magic bytes) trước khi Feed/Friend lưu URL ảnh. Không nhận body binary, không phải LiveKit media-server, không xử lý video (Short Video có VideoStoragePort riêng).

Owns: bảng image_assets (ownerUserId, status pending/ready, purpose, contentType, sizeBytes, storageKey); object key quarantine rồi key final bất biến trên S3/R2/MinIO

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/media/images/upload-intent | jwt | Xin presigned PUT URL + assetId (purpose post/message/story); kiểm content type cho phép, sizeBytes <= MEDIA_IMAGE_MAX_BYTES, chặn guest. |

## Coordination edges (why this service talks to others)

### image_assets: owner/status (c1) with postgres [infra, out, shared-db]

- endpoints: e1
- what: Ghi hàng image_assets khi cấp intent (status pending, ownerUserId, storageKey quarantine) và đọc/cập nhật khi resolveImageUrl (kiểm owner + purpose, chuyển sang ready với key final).
- why: Lúc gắn ảnh vào Post/Story/Chat phải chứng minh asset thuộc đúng caller, đúng purpose và đã được kiểm tra; trạng thái đó chỉ bền vững nếu nằm trong DB, không thể tin client.
- if missing: Không xác thực được quyền sở hữu asset: user có thể dùng assetId của người khác hoặc asset chưa upload/chưa kiểm tra.
- evidence: apps/core-api/src/modules/media/entities/image-asset.entity.ts:6
- review: sound — image_assets lưu ownerUserId, purpose, status pending; resolveImageUrl lọc theo owner và purpose rồi set ready. Với provider dev không kiểm object.

### resolveImageUrl (Post/Story) (c2) with feed [service, in, di-call]

- endpoints: e1
- what: FeedController (đăng Post) và StoryController (đăng Story) gọi MediaService.resolveImageUrl(userId, assetId, purpose) để đổi assetId thành URL final đã kiểm chứng trước khi tạo nội dung.
- why: DTO của Post/Story chỉ nhận assetId chứ không nhận URL; việc đổi sang URL phải đi qua Media để kiểm owner/purpose/status/object trên storage, nên Feed bắt buộc phối hợp với Media ở bước tạo nội dung.
- if missing: Feed phải tin URL do client gửi, bỏ qua kiểm tra chủ sở hữu và quarantine: có thể gắn ảnh người khác hoặc URL ngoài kiểm soát.
- evidence: apps/core-api/src/modules/feed/controllers/story.controller.ts:57, apps/core-api/src/modules/feed/feed.controller.ts:66
- review: sound — Story/Feed controller gọi resolveImageUrl với đúng purpose; guest bỏ qua bước resolve và bị service từ chối.

### resolveImageUrl (chat) (c3) with friend [service, in, di-call]

- endpoints: e1
- what: FriendController gọi MediaService.resolveImageUrl cho ảnh gửi trong chat 1-1 (purpose message) trước khi lưu message.
- why: Ảnh chat cũng chỉ được nhận qua assetId đã upload của chính người gửi, cùng một chốt chặn với Post/Story để không có đường tắt nhận URL tuỳ ý.
- if missing: Người dùng có thể nhét URL ảnh bất kỳ vào tin nhắn, vượt qua kiểm soát nội dung/quyền sở hữu.
- evidence: apps/core-api/src/modules/friend/friend.controller.ts:210
- review: sound — FriendController dùng ImageAssetPurpose.Message; cùng chốt chặn owner/purpose với Post/Story.

### presigned PUT + promote (c50) with external:object-storage [external, out, sdk]

- endpoints: e1
- what: Ký presigned PUT URL (getSignedUrl + PutObjectCommand) cho key quarantine; khi resolveImageUrl: head + readPrefix(4096 byte) kiểm size/MIME/magic bytes rồi CopyObject sang key final xác định images/{owner}/{asset} và xoá key cũ (S3/R2/MinIO theo MEDIA_STORAGE_PROVIDER; provider dev bỏ qua kiểm tra).
- why: Body ảnh không được đi qua core-api (tốn băng thông, dễ bị DoS) nên storage giữ file; nhưng chỉ storage mới biết object có thật, kích thước và nội dung thật, nên core-api bắt buộc hỏi lại storage trước khi công bố URL final.
- if missing: Client phải gửi URL ảnh tuỳ ý hoặc core-api phải nhận binary: mở đường cho ảnh độc hại/URL ngoài kiểm soát và làm core-api nghẽn vì upload.
- evidence: apps/core-api/src/modules/media/ports/s3-image-storage.provider.ts:101, apps/core-api/src/modules/media/media.service.ts:114
- review: corrected — Key final chỉ xác định chứ không bất biến (không có object-lock); provider dev bỏ qua head/magic bytes. head, readPrefix và Copy rồi Delete đã xác nhận.

### xin intent rồi PUT thẳng (c51) with web [client, in, http]

- endpoints: e1
- what: apps/web gọi POST upload-intent, rồi putPresignedFile PUT file thẳng tới uploadUrl với đúng Content-Type đã khai báo, cuối cùng gửi assetId (không gửi URL) vào endpoint Post/Story/Chat.
- why: Presigned URL chỉ ký Content-Type nên client phải PUT đúng type đã khai báo; sizeBytes không được ký nên kích thước/MIME/magic bytes chỉ bị chặn khi resolveImageUrl (ASSET_NOT_READY 422). core-api chỉ tin assetId nên client phải làm đủ 3 bước: xin intent, PUT file, gắn assetId.
- if missing: Không có file trên storage nên resolveImageUrl trả 422; hoặc client gửi URL tự do bị DTO từ chối.
- evidence: apps/web/src/shared/media/image-upload.ts:44
- review: corrected — Size không nằm trong chữ ký presigned; storage không từ chối theo size, resolveImageUrl mới chặn bằng ASSET_NOT_READY 422.

## Notes

- Short Video KHÔNG dùng module Media: nó có VideoStoragePort riêng và endpoint upload-intent riêng (short-video.controller.ts).
- Với MEDIA_STORAGE_PROVIDER=dev, resolveImageUrl bỏ qua head/promote/kiểm magic bytes và chỉ đổi storageKey sang key final rồi đánh dấu Ready (media.service.ts:119-125); provider dev bị chặn ở production nên kiểm tra thật chỉ có với r2/s3/minio.
- resolveImageUrl được controller Feed/Story/Friend gọi TRƯỚC khi tạo nội dung và nằm ngoài transaction tạo nội dung: asset có thể chuyển sang Ready/đã promote rồi mới tạo nội dung thất bại, asset đó vẫn còn nhưng chưa gắn vào đâu.
- createUploadIntent chặn guest (assertNotGuest); story controller cũng bỏ qua ảnh nếu user là guest.
- Doc media-service.md khớp code ở các điểm chính (quarantine, magic bytes, promote); không thấy lệch.
