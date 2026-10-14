# Relay outbox của Economy ra Kafka

id: flow:economy-outbox-relay
group: Tiền tệ & quà
services: module:economy

Economy ghi OutboxEvent cùng giao dịch sổ cái; một job nền định kỳ đọc các dòng chưa gửi, publish lên Kafka rồi đánh dấu đã gửi, nên cần phối hợp giữa PostgreSQL (khóa dòng) và Kafka (broker).

Job relay khóa lô OutboxEvent trong PostgreSQL, publish từng event lên Kafka, chuyển DLQ khi quá số lần thử rồi lưu trạng thái.

## Participants

- economy (service): Economy (OutboxRelayService)
- postgres (infra): PostgreSQL
- kafka (infra): Kafka

## Steps

1. economy -> postgres [call/shared-db]: Mở transaction, chọn tối đa 100 event chưa publish bằng FOR UPDATE SKIP LOCKED (Job chạy theo setInterval khi ECONOMY_OUTBOX_RELAY_ENABLED bật; runExclusive bỏ qua tick nếu vòng trước còn chạy. Lọc published_at IS NULL và dead_lettered_at IS NULL, sắp theo created_at.) {apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:67, apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:72}
2. postgres -> economy [return/shared-db]: Trả lô OutboxEvent; nếu rỗng thì kết thúc vòng, trả 0
3. economy -> kafka [async/kafka]: Publish từng event (key = transactionId/userId/id) lên topic của event (Thành công thì set publishedAt và xóa lastError; lỗi thì tăng attempts, lưu lastError và ghi log, các event khác vẫn tiếp tục.) {apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:82, apps/core-api/src/common/events/kafka-event-bus.adapter.ts:109}
4. economy -> kafka [async/kafka]: Khi attempts đạt ngưỡng tối đa thì publish sang topic dead-letter (.DLQ) (Nếu publish DLQ cũng lỗi thì chỉ ghi log; finally vẫn đặt deadLetteredAt để dòng không bị thử lại vô hạn và còn lại làm nguồn replay.) {apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:117, apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:144}
5. economy -> postgres [call/shared-db]: Lưu cả lô (publishedAt, attempts, lastError, deadLetteredAt) rồi commit {apps/core-api/src/modules/economy/jobs/outbox-relay.service.ts:103}

## Invariants

- Khóa FOR UPDATE SKIP LOCKED giữ đến khi lưu xong, nên nhiều instance chạy song song lấy các lô khác nhau và không publish trùng cùng một dòng.
- Publish diễn ra trước khi đánh dấu publishedAt: nếu process chết giữa chừng thì transaction rollback và event được gửi lại, tức at-least-once; consumer phải dedupe theo id event nằm trong message.
- Lỗi publish được bắt trong từng event nên một event hỏng không chặn cả lô; chỉ khi vượt ECONOMY_OUTBOX_MAX_ATTEMPTS mới chuyển DLQ và đánh dấu terminal.

## Failure modes

- when Kafka lỗi khi publish một event: attempts tăng, lastError được lưu, dòng vẫn chưa publish và được thử lại ở vòng sau cho đến ngưỡng tối đa.
- when Publish DLQ cũng lỗi: Chỉ ghi log; dòng vẫn được đặt deadLetteredAt và giữ lại trong bảng làm nguồn replay cho vận hành.
- when Vòng trước còn chạy khi tick kế đến: runExclusive trả 0 và bỏ qua tick, không chồng vòng trong cùng process.

## Endpoints covered

- OutboxRelayService.onApplicationBootstrap [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
