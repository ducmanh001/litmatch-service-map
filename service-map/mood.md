# Service map: Mood

id: service-map:mood
module: module:mood
deployable: core-api
confidence: high

Trạng thái tâm trạng preset, hết hạn theo TTL

Sở hữu catalog mood_presets và log append-only mood_status_events (set/clear), suy ra mood hiện tại lúc đọc dựa trên dòng mới nhất và expiresAt. Không sở hữu block/report (Safety), không có free-text/moderation, không phát realtime và không có job dọn.

Owns: mood_presets (code unique, label, emoji, active, sortOrder); mood_status_events (append-only; idempotencyKey unique; kind set|clear)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/mood/presets | jwt | Liệt kê preset mood đang active (sortOrder, code) để client dựng UI chọn; chỉ cần JWT. |
| e2 | rest | GET | /api/v1/mood/status/me | jwt | Mood hiện tại của chính mình, suy ra từ dòng event mới nhất; không qua kiểm tra block. |
| e3 | rest | GET | /api/v1/mood/status/:userId | jwt | Mood công khai của userId khác theo góc nhìn người xem; trả rỗng nếu giữa hai người có block active 2 chiều (SafetyService.getBlockedUserIds). |
| e4 | rest | POST | /api/v1/mood/status | jwt | Đặt mood theo presetCode đang active, bắt buộc header Idempotency-Key; mỗi lần gọi (kể cả cùng preset) là 1 dòng event mới và refresh TTL MOOD_STATUS_TTL_HOURS. |
| e5 | rest | DELETE | /api/v1/mood/status | jwt | Tắt mood bằng cách ghi dòng kind='clear' (không xoá dòng cũ), bắt buộc Idempotency-Key, trả 204. |

## Coordination edges (why this service talks to others)

### ẩn mood khi bị block (c1) with safety [service, out, di-call]

- endpoints: e3
- what: getPublicMood gọi SafetyService.getBlockedUserIds(viewerId) mỗi lần đọc rồi trả null nếu targetId nằm trong danh sách block active 2 chiều (bỏ qua nếu viewerId === targetId).
- why: Block là dữ liệu của Safety (bảng blocks append-only, trạng thái mới nhất mỗi cặp quyết định). Người đã block không được thấy mood của nhau, và phải kiểm tra lại mỗi lần đọc (docs/10 § 10.0.C) vì block/unblock có thể xảy ra bất cứ lúc nào; chỉ xét block, không xét report (khác getHiddenUserIds của Discovery).
- if missing: Người đã bị block vẫn xem được mood của người block mình qua GET /mood/status/:userId, vô hiệu hoá tính năng block.
- evidence: apps/core-api/src/modules/mood/mood.service.ts:131
- review: sound

### preset + event mood (c2) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5
- what: Đọc mood_presets (find/findOneBy active) và ghi mood_status_events; idempotencyKey (prefix theo kind + userId) có unique ở DB, bắt unique violation để đọc lại dòng cũ.
- why: Mood hiện tại = dòng event mới nhất nên log phải bền vững và append-only; unique idempotencyKey ở DB là chốt chặn retry/double-click tạo trùng dòng mà không cần lock ở app. Preset nằm ở DB để chỉnh qua seed/migration thay vì hardcode.
- if missing: Retry mạng tạo nhiều dòng set/clear trùng lặp, mất lịch sử hoặc mood nhảy trạng thái; không có catalog thì không validate được presetCode.
- evidence: apps/core-api/src/modules/mood/entities/mood-preset.entity.ts:7, apps/core-api/src/modules/mood/entities/mood-status-event.entity.ts:22
- review: sound

## Notes

- Doc nói MoodService.getPublicMood là API composition 'dùng ở friend list, discovery card, profile qua DI': MoodModule có exports [MoodService] nhưng không có module nào khác import/gọi MoodService (dossier: không có caller vào). Hiện chỉ endpoint GET /mood/status/:userId dùng nó.
- Doc yêu cầu không dùng ở card ẩn danh trước-match Soul Match; đúng là không có chỗ nào gọi, nhưng đó là kỷ luật call site, không có cờ nào chặn trong service (doc cũng nói vậy).
- Không tìm thấy client apps/web hay admin gọi /mood/* trong source (grep mood/status không có kết quả), nên không thêm collaborator client.
- Hết hạn mood là derive-khi-đọc (deriveCurrentMood so expiresAt với giờ server lúc đọc), không có cron/job.
