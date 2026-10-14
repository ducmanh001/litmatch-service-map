# Kiểm soát truy cập media: LiveKit và upload file?

id: qa:media-access
category: authz
services: module:party-room, module:calling, module:media, module:short-video, module:media-server

Quyền vào phòng media là capability token ngắn hạn do core-api ký (LiveKit grants theo vai trò); đổi vai trò giữa chừng thì core-api đẩy quyền xuống SFU chứ không tin client. File ảnh hoặc video upload bằng presigned URL thẳng lên object storage, ảnh còn qua quarantine rồi mới promote.

### Capability token

Người nghe nhận token canPublish=false nên tự bật mic bị LiveKit chặn; cấp hoặc thu quyền nói phải đẩy xuống SFU bằng updateParticipant.

services: party-room, calling, media-server; evidence: apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:24, apps/core-api/src/modules/calling/calling.service.ts:191

### Presigned upload

core-api chỉ cấp URL ký sẵn nên body ảnh hoặc video không đi qua; ảnh còn nằm ở key quarantine tới khi kiểm MIME và kích thước rồi mới promote sang key công khai.

services: media, short-video; evidence: apps/core-api/src/modules/media/media.controller.ts:24, apps/core-api/src/modules/media/media.service.ts:114, apps/core-api/src/modules/short-video/dto/short-video.dtos.ts:19

## Alternatives not chosen

- Cho client giữ quyền và tự khai trong phòng: Client có thể bị sửa; quyền phải được enforce ở SFU bằng grant do server ký.
