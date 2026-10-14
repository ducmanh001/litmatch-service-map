# Trạng thái tâm trạng và quyền xem

id: flow:mood-status-visibility
group: Ghép đôi & gọi
services: module:mood, module:safety

Người dùng đặt và xoá mood theo preset với Idempotency-Key, rồi người khác xem mood đó. Mood sở hữu log sự kiện, còn quyền xem phụ thuộc block do Safety sở hữu nên getPublicMood phải hỏi Safety trước khi đọc mood.

Đặt/xoá mood ghi log idempotent vào PostgreSQL; xem mood của mình chỉ suy ra từ log, còn xem mood người khác hỏi Safety về block hai chiều trước khi suy ra mood hiện tại lúc đọc.

## Participants

- web (client): Web client
- mood (service): Mood
- safety (service): Safety
- postgres (infra): PostgreSQL

## Steps

1. web -> mood [call/http]: GET /mood/presets lấy catalog preset đang bật
2. mood -> postgres [call/shared-db]: Đọc mood_presets đang active, sắp theo sortOrder {apps/core-api/src/modules/mood/mood.service.ts:47}
3. web -> mood [call/http]: POST /mood/status với presetCode và Idempotency-Key
4. mood -> postgres [call/shared-db]: Kiểm preset tồn tại và active, tính expiresAt theo TTL {apps/core-api/src/modules/mood/mood.service.ts:63}
5. mood -> postgres [call/shared-db]: Ghi dòng mood_status_events kind set; trùng key thì đọc lại dòng cũ (Unique violation trên idempotencyKey được coi là replay hợp lệ: findOneByOrFail trả dòng đã có.) {apps/core-api/src/modules/mood/mood.service.ts:167}
6. mood -> web [return/http]: Trả mood hiện tại gồm preset, setAt, expiresAt
7. web -> mood [call/http]: GET /mood/status/me xem mood của chính mình (Route cố định 'status/me' được khai báo trước 'status/:userId' trong controller.) {apps/core-api/src/modules/mood/mood.controller.ts:47}
8. mood -> postgres [call/shared-db]: Suy ra mood của mình từ event mới nhất, không kiểm block (getMyMood gọi thẳng deriveCurrentMood, không hỏi Safety vì không tự block chính mình. Trả null nếu event mới nhất là clear hoặc đã quá expiresAt.) {apps/core-api/src/modules/mood/mood.service.ts:115}
9. mood -> web [return/http]: Trả mood hiện tại của mình hoặc null
10. web -> mood [call/http]: GET /mood/status/:userId xem mood của người khác
11. mood -> safety [call/di-call]: Hỏi getBlockedUserIds(viewerId) để kiểm block hai chiều (Nếu viewerId trùng targetId thì getPublicMood bỏ qua bước hỏi Safety. getBlockedUserIds trả userId có block ACTIVE hai chiều (dòng mới nhất mỗi cặp là Blocked).) {apps/core-api/src/modules/mood/mood.service.ts:131}
12. safety -> postgres [call/shared-db]: Đọc bảng blocks theo hai chiều, lấy hành động mới nhất {apps/core-api/src/modules/safety/safety.service.ts:241}
13. safety -> mood [return/di-call]: Trả danh sách userId đang block với viewer
14. mood -> postgres [call/shared-db]: Nếu không bị block: đọc event mới nhất và preset để suy ra mood (Nếu targetId nằm trong danh sách block thì trả null, không đọc event. Mood hết hạn hoặc bị clear cũng trả null.) {apps/core-api/src/modules/mood/mood.service.ts:142}
15. mood -> web [return/http]: Trả mood hoặc null nếu bị block, hết hạn, đã xoá
16. web -> mood [call/http]: DELETE /mood/status với Idempotency-Key
17. mood -> postgres [call/shared-db]: Ghi dòng mood_status_events kind clear, idempotent theo key {apps/core-api/src/modules/mood/mood.service.ts:103}
18. mood -> web [return/http]: Trả 204 No Content

## Invariants

- Mood là log append-only; mood hiện tại được suy ra lúc đọc từ dòng mới nhất và expiresAt, nên không cần job dọn hết hạn.
- Quyền xem thuộc Safety: Mood phải hỏi getBlockedUserIds mỗi lần đọc (không cache) để block mới có hiệu lực ngay, và không tự lưu block.
- Idempotency-Key được gói thành khoá unique trong DB nên retry cùng key không tạo thêm dòng mà đọc lại dòng cũ.
- Xem mood của chính mình không qua Safety: getMyMood gọi thẳng deriveCurrentMood, chỉ xem mood người khác mới kiểm block.

## Failure modes

- when Preset không tồn tại hoặc đã tắt khi set mood: Ném DomainException PRESET_NOT_FOUND với HTTP 404, không ghi event.
- when Safety.getBlockedUserIds lỗi khi xem mood người khác: Lỗi lan lên getPublicMood (không có try/catch), request xem mood thất bại thay vì lộ mood.
- when Retry set/clear với cùng Idempotency-Key: Unique violation được bắt và trả dòng đã tồn tại, không tạo bản ghi mới.

## Endpoints covered

- GET /mood/presets [module:mood] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mood.md
- GET /mood/status/me [module:mood] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mood.md
- GET /mood/status/:userId [module:mood] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mood.md
- POST /mood/status [module:mood] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mood.md
- DELETE /mood/status [module:mood] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/mood.md
