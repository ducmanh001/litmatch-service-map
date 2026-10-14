# Job dọn lời mời và vé ghép đôi hết hạn

id: flow:matching-sweeper-jobs
group: Ghép đôi & gọi
services: module:matching

Hai worker nền của matching chạy theo timer trên mọi replica, nên phải dùng advisory lock Postgres để chỉ một replica làm việc; kết quả cuối cùng phải đồng bộ giữa bảng Postgres và hàng đợi Redis.

Hai sweeper giành advisory lock (connection riêng), cập nhật invite/ticket hết hạn trong Postgres rồi đồng bộ hàng đợi Redis sau khi commit từng session.

## Participants

- matching (service): Matching (sweeper)
- postgres (infra): PostgreSQL
- redis (infra): Redis (hàng đợi)

## Steps

1. matching -> matching [async/config]: InviteSweeper đăng ký interval khi bootstrap, mỗi tick gọi runOnce (Hai job (invite, ticket) có timer riêng, độc lập nhau; thứ tự vẽ ở đây chỉ để dễ đọc. Chu kỳ lấy từ config MATCHING_INVITE_SWEEPER_INTERVAL_MS.) {apps/core-api/src/modules/matching/jobs/invite-sweeper.service.ts:40}
2. matching -> postgres [call/shared-db]: Mở transaction, pg_try_advisory_xact_lock; không giành được thì rollback và nghỉ (Lock nằm trong transaction của một QueryRunner riêng, giữ suốt lúc task chạy. Replica không giành được lock đặt clusterRetryAfter (followerRetryMs, mặc định max(intervalMs, 5s)) và bỏ qua các tick tới; job còn skipWhenIdle khi process nhàn rỗi.) {apps/core-api/src/common/scheduling/managed-interval.ts:128}
3. matching -> postgres [call/shared-db]: UPDATE match_invites Pending quá expires_at thành Expired (batch 200) (UPDATE chạy bằng dataSource.query (connection khác, autocommit), KHÔNG nằm trong transaction giữ lock; lock chỉ bảo đảm một replica chạy tại một thời điểm.) {apps/core-api/src/modules/matching/jobs/invite-sweeper.service.ts:63}
4. matching -> matching [async/config]: TicketSweeper đăng ký interval khi bootstrap, mỗi tick gọi runOnce {apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:63}
5. matching -> postgres [call/shared-db]: Giành advisory lock riêng của job matching-ticket-sweeper {apps/core-api/src/common/scheduling/managed-interval.ts:128}
6. matching -> postgres [call/shared-db]: UPDATE match_tickets queued quá hạn thành expired, RETURNING shard (UPDATE có điều kiện status='queued' nên không expire nhầm ticket vừa được matcher chuyển sang matched.) {apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:105}
7. matching -> redis [call/redis-state]: ZREM từng ticket hết hạn khỏi sorted set của shard (idempotent) {apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:133}
8. matching -> postgres [call/shared-db]: Chọn session pending_confirm quá MATCHING_CONFIRM_TIMEOUT_SECONDS {apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:146}
9. matching -> postgres [call/shared-db]: Mỗi session một transaction: khóa session và ticket, expire, tạo ticket requeue (Re-check status dưới lock; bên đã confirm được requeue bằng ticket MỚI với idempotency key tất định, bên chưa confirm thì expired.) {apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:171}
10. matching -> redis [call/redis-state]: Sau commit: ZADD NX ticket requeue và đánh dấu shard active (Không có try/catch quanh Redis: nếu Redis lỗi, lỗi ném ra khỏi expireSession và làm tick dừng; ticket requeue chỉ còn trong DB và sẽ bị sweeper expire khi quá hạn (theo comment trong code).) {apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:228, apps/core-api/src/modules/matching/jobs/ticket-sweeper.service.ts:234}

## Invariants

- Advisory lock theo jobName (pg_try_advisory_xact_lock giữ trên QueryRunner riêng) bảo đảm chỉ một replica chạy mỗi sweeper tại một thời điểm; lock không bao các UPDATE (chạy autocommit trên connection khác).
- Redis chỉ được cập nhật SAU khi transaction DB commit, để hàng đợi không bao giờ chứa ticket mà DB chưa ghi nhận.
- Mỗi session expire trong transaction riêng, khóa session rồi tickets, và re-check status dưới lock để không đè lên confirm vừa hoàn tất.

## Failure modes

- when Redis lỗi sau khi DB đã commit requeue: Lỗi không bị nuốt: enqueue/markActive ném ra, tick dừng (các session còn lại chờ tick sau); ticket requeue tồn tại trong DB nhưng không có trong hàng đợi, sweeper sẽ expire khi quá hạn.
- when Tick ném lỗi trong lúc giữ advisory lock: Transaction chứa lock bị rollback (chỉ nhả lock, không hoàn tác UPDATE đã autocommit) và replica đặt clusterRetryAfter rồi thử lại sau followerRetryMs.

## Endpoints covered

- InviteSweeperService.onApplicationBootstrap [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
- TicketSweeperService.onApplicationBootstrap [module:matching] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/matching.md
