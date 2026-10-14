# Xem, huỷ và tăng tốc vé ghép đôi

id: flow:matching-ticket-status-cancel-speedup
group: Ghép đôi & gọi
services: module:matching, module:economy

Người dùng đọc trạng thái vé và giá tăng tốc theo hạng VIP, huỷ vé đang xếp hàng, hoặc trả DIA để tăng ưu tiên. Matching sở hữu vé và hàng đợi Redis, còn hạng VIP và việc trừ DIA thuộc Economy nên hai service phải phối hợp.

Ba thao tác trên vé: đọc (current/:id), huỷ (DELETE), tăng tốc (speedup) và cách Matching gọi Economy, Postgres, Redis.

## Participants

- web (client): Web client
- matching (service): Matching
- economy (service): Economy
- postgres (infra): PostgreSQL
- redis (infra): Redis

## Steps

1. web -> matching [call/http]: GET /matching/tickets/current để phục hồi vé sau reload
2. matching -> postgres [call/shared-db]: Tìm vé queued/matched mới nhất của user {apps/core-api/src/modules/matching/matching.service.ts:247}
3. matching -> economy [call/di-call]: Lấy hạng VIP đang hiệu lực để tính giá tăng tốc (Economy đọc ví (wallets) và trả activeVipTier; Matching chọn giá config theo SVIP/VIP/thường.) {apps/core-api/src/modules/matching/matching.service.ts:264}
4. matching -> web [return/http]: Trả vé hiện tại kèm giá tăng tốc
5. web -> matching [call/http]: GET /matching/tickets/:id để poll trạng thái vé
6. matching -> postgres [call/shared-db]: Đọc vé theo id rồi kiểm tra vé thuộc về người gọi {apps/core-api/src/modules/matching/matching.service.ts:229}
7. matching -> economy [call/di-call]: Lấy hạng VIP để tính giá tăng tốc trả về {apps/core-api/src/modules/matching/matching.service.ts:264}
8. matching -> web [return/http]: Trả trạng thái vé kèm giá tăng tốc
9. web -> matching [call/http]: DELETE /matching/tickets/:id để huỷ vé
10. matching -> postgres [call/shared-db]: Khoá vé, kiểm tra chủ vé và chuyển trạng thái, lưu cancelled (Khoá pessimistic_write; assertOwnership và assertTransition chạy trong callback transaction.) {apps/core-api/src/modules/matching/matching.service.ts:291}
11. matching -> redis [call/redis-state]: ZREM vé khỏi hàng đợi shard sau khi commit (Lỗi ZREM sẽ ném ra lỗi (có await); comment trong code nói vé thành zombie Redis và matcher kiểm tra lại lúc ghép.) {apps/core-api/src/modules/matching/matching.service.ts:299, apps/core-api/src/modules/matching/redis/redis-matching-queue.adapter.ts:22}
12. matching -> web [return/http]: Trả vé cancelled kèm giá tăng tốc (đọc VIP từ Economy) (Controller gọi getSpeedupPriceDiamondForUser, tức economy.getActiveVipTier, để dựng TicketDto.)
13. web -> matching [call/http]: POST /matching/tickets/:id/speedup kèm Idempotency-Key
14. matching -> postgres [call/shared-db]: Đọc vé, kiểm tra chủ vé và phải đang queued {apps/core-api/src/modules/matching/matching.service.ts:229}
15. matching -> economy [call/di-call]: Lấy hạng VIP để xác định giá DIA {apps/core-api/src/modules/matching/matching.service.ts:264}
16. matching -> economy [call/di-call]: spendDiamond MatchingSpeedup với idempotency key có tiền tố (Economy ghi sổ kép qua ledger.record (nợ ví user, có doanh thu hệ thống); replayed=true nghĩa là đã trừ trước đó.) {apps/core-api/src/modules/matching/matching.service.ts:448, apps/core-api/src/modules/economy/economy.service.ts:484}
17. matching -> postgres [call/shared-db]: Cộng dồn priorityBoostMs nếu chưa replay, rồi đọc lại vé (increment chỉ khớp khi vé còn queued.) {apps/core-api/src/modules/matching/matching.service.ts:458}
18. matching -> redis [call/redis-state]: ZADD XX đặt lại score tuyệt đối từ tổng boost trong DB (XX: member đã rời hàng đợi thì bỏ qua; replay cùng key chỉ sửa score về đúng tổng đã trả.) {apps/core-api/src/modules/matching/matching.service.ts:467, apps/core-api/src/modules/matching/redis/redis-matching-queue.adapter.ts:18}
19. matching -> web [return/http]: Trả transactionId, replayed và vé với giá VIP mới đọc

## Invariants

- Giá tăng tốc phụ thuộc hạng VIP nên Matching luôn hỏi Economy mỗi lần trả vé, không tự giữ trạng thái VIP; Economy là nơi duy nhất sở hữu ví.
- Matching không ghi sổ: việc trừ DIA đi qua economy.spendDiamond (ledger.record), Matching chỉ làm hiệu ứng boost vé và hàng đợi.
- Boost được cộng vào DB trước, rồi ZADD XX đặt score tuyệt đối từ tổng DB nên retry cùng key không boost đôi.
- Huỷ vé commit trạng thái trong Postgres trước rồi mới ZREM Redis, vì Postgres là nguồn sự thật của vé.

## Failure modes

- when ZREM Redis lỗi sau khi vé đã cancelled trong DB: Lỗi được ném ra từ cancelTicket; vé thành zombie trong Redis, matcher kiểm tra lại lúc ghép sẽ loại (theo comment trong code).
- when Vé không còn queued khi speedup: Trả lỗi TICKET_INVALID_TRANSITION (409) trước khi trừ DIA.
- when Vé không thuộc user gọi: assertOwnership ném TICKET_FORBIDDEN (403).
- when Vé vừa được ghép giữa lúc trừ DIA và increment: DIA vẫn bị trừ, increment không khớp do status không còn queued, không hoàn tiền (theo comment code).

## Endpoints covered

- GET /matching/tickets/current [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- GET /matching/tickets/:id [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- DELETE /matching/tickets/:id [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- POST /matching/tickets/:id/speedup [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
