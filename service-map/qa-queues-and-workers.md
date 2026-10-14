# Hàng đợi và worker nền: dùng gì, ở đâu?

id: qa:queues-and-workers
category: messaging
services: module:matching, module:economy, module:friend, module:party-room

Không có thư viện job queue chung (không BullMQ, Agenda hay pg-boss trong package.json). Ghép đôi dùng Redis sorted set chia shard (ZADD/ZPOPMIN) rồi xác minh lại trong PostgreSQL bằng SELECT FOR UPDATE; worker nền là vòng lặp ManagedInterval chạy singleton nhờ advisory lock; sự kiện nghiệp vụ bền đi qua outbox rồi Kafka (có DLQ).

### Hàng đợi ghép đôi

Mỗi shard là một sorted set: ZADD thêm vé, ZPOPMIN lấy vé theo điểm một cách nguyên tử nên hai pod không lấy trùng; Redis không phải nguồn sự thật nên matcher khoá hai ticket trong PostgreSQL (pessimistic_write) và verify lại trước khi tạo session.

services: matching; evidence: apps/core-api/src/modules/matching/redis/redis-matching-queue.adapter.ts:18, apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:237

### Worker nền singleton

ManagedInterval chạy worker và sweeper theo chu kỳ trong process và dùng pg advisory lock để chỉ một replica chạy tại một thời điểm.

services: matching, friend, party-room; evidence: apps/core-api/src/common/scheduling/managed-interval.ts:128

### Outbox rồi Kafka

Economy ghi OutboxEvent cùng transaction nghiệp vụ; OutboxRelayService lấy hàng bằng FOR UPDATE SKIP LOCKED rồi publish lên Kafka; adapter có DLQ cho tin lỗi.

services: economy; evidence: apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:72, apps/core-api/src/common/events/event-bus.port.ts:23

## Alternatives not chosen

- BullMQ hoặc job queue trên Redis: Suy luận: thêm một lớp hạ tầng và trạng thái thứ hai khi trạng thái thật đã nằm ở PostgreSQL và worker chỉ là vòng lặp đơn giản.
- Redis list làm hàng đợi ghép đôi: Không có điểm để lấy theo thứ tự ưu tiên; sorted set cho ZPOPMIN theo điểm.

## Caveats

- Trong profile hosted-free relay Kafka bị tắt nhưng outbox vẫn được ghi atomic (docs/03 § 3.9).
