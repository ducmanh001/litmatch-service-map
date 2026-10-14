# Chống xử lý trùng (idempotency) làm thế nào?

id: qa:idempotency
category: consistency
services: module:matching, module:support, module:economy, module:gift

Hai lớp: client gửi Idempotency-Key (decorator @IdempotencyKey) và DB giữ chốt cuối bằng unique trên cột idempotency_key, nên retry mạng nhận lại kết quả cũ thay vì tạo bản ghi hoặc trừ tiền hai lần.

### Header và decorator

Controller nhận Idempotency-Key qua một decorator dùng chung rồi truyền xuống service.

services: matching, gift, support; evidence: apps/core-api/src/common/decorators/idempotency-key.decorator.ts:18

### Unique ở DB

Ticket ghép đôi có idempotencyKey unique, ticket support có unique (userId, idempotencyKey), Transaction của Economy có idempotencyKey unique nên kiểm-rồi-ghi không bị race.

services: matching, support, economy; evidence: apps/core-api/src/modules/matching/entities/match-ticket.entity.ts:127, apps/core-api/src/modules/support/entities/support-ticket.entity.ts:21

## Alternatives not chosen

- Chỉ check-rồi-insert ở tầng code: Vẫn dính race khi hai request chạy song song; docs/03 yêu cầu unique ở tầng DB.
