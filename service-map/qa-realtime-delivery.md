# Realtime dùng Redis Streams, queue hay pub/sub?

id: qa:realtime-delivery
category: messaging
services: module:signaling-gateway, module:matching, module:gift, module:friend, module:party-room, module:soul-match

Redis pub/sub fanout, at-most-once, không lưu và không replay, chỉ publish sau khi commit. Không dùng Redis Streams hay hàng đợi cho realtime vì sự kiện chỉ là gợi ý, REST polling mới là nguồn sự thật.

### Đường đi

core-api PUBLISH lên kênh realtime:user:{userId} với payload đã tính sẵn cho từng người nhận; signaling-gateway PSUBSCRIBE realtime:user:* rồi đẩy xuống room của user.

services: signaling-gateway, gift, friend; evidence: apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11, apps/signaling-gateway/src/app/signaling.gateway.ts:14

### Ngữ nghĩa best-effort

Publish bọc try/catch và chỉ log khi lỗi để không phá nghiệp vụ đã commit; client luôn còn REST polling làm đường dự phòng.

services: matching, soul-match; evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:370, docs/services/realtime-gateway.md:38

### Nhiều instance

Socket.IO Redis adapter đồng bộ việc phát tin giữa nhiều instance gateway; client cố định transport websocket nên không cần sticky session.

services: signaling-gateway; evidence: apps/signaling-gateway/src/app/redis-adapter.service.ts:46

## Alternatives not chosen

- Redis Streams: Bền, có consumer group và replay nhưng cần ack và quản lý offset; thừa với tin chỉ là gợi ý refetch.
- Outbox + Kafka cho realtime: docs/services/realtime-gateway.md chọn không dùng outbox cho luồng này vì sự kiện ephemeral và đã có fallback.
- Hàng đợi (list) theo từng user: Suy luận: không replay đúng nghĩa và phải dọn khi user offline.

## Caveats

- Đã quét toàn bộ apps/core-api/src và apps/signaling-gateway/src: không có lệnh XADD, XREAD, XREADGROUP, XACK hay LPUSH, BRPOP nào.
