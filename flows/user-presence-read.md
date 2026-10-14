# Xem trạng thái online của người dùng

id: flow:user-presence-read
group: Danh tính
services: module:user, module:signaling-gateway

Khi một người dùng xem trạng thái online của người khác, User đọc cài đặt riêng tư trong PostgreSQL rồi đọc khoá presence trong Redis do lease của signaling-gateway ghi, nên User phụ thuộc dữ liệu do gateway sở hữu.

GET /users/:id/presence kết hợp cài đặt riêng tư (Postgres) với khoá presence Redis do signaling-gateway ghi.

## Participants

- web (client): Web client
- user (service): User
- postgres (infra): PostgreSQL
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway

## Steps

1. web -> user [call/http]: GET /users/:id/presence xem trạng thái online (Route được bảo vệ bởi JwtAuthGuard toàn cục.) {apps/core-api/src/app/app.module.ts:154}
2. user -> postgres [call/shared-db]: Đọc user_privacy_settings của người bị xem (showOnlineStatus) {apps/core-api/src/modules/user/services/privacy-settings.service.ts:21}
3. postgres -> user [return/shared-db]: Trả cài đặt riêng tư
4. signaling-gateway -> redis [async/redis-state]: Từ trước: lease kết nối ghi/gia hạn khoá presence (acquire/refresh/release chạy script Lua trên khoá realtimePresenceKey(userId); gateway sở hữu dữ liệu này.) {apps/signaling-gateway/src/app/connection-quota.service.ts:130}
5. user -> redis [call/redis-state]: Xoá lease hết hạn rồi đếm khoá presence (zcard > 0) (Chỉ cần thực hiện khi showOnlineStatus bật (toán tử && ngắt sớm); lỗi Redis trả false (fail-closed).) {apps/core-api/src/modules/user/redis/redis-presence-reader.adapter.ts:16}
6. redis -> user [return/redis-state]: Trả số lease còn hiệu lực
7. user -> web [return/http]: Trả UserPresenceDto { isOnline }

## Invariants

- Cài đặt riêng tư được đọc trước: nếu showOnlineStatus tắt thì isOnline luôn false và Redis không bị hỏi, nên quyền riêng tư của User luôn thắng dữ liệu presence của gateway.
- Khoá presence chỉ do lease adapter của signaling-gateway ghi; User chỉ đọc qua cổng UserPresenceReaderPort nên không cần gọi trực tiếp gateway.

## Failure modes

- when Redis lỗi khi đọc presence: RedisPresenceReader bắt lỗi và trả false, người dùng hiển thị offline thay vì lộ trạng thái sai (fail-closed).
- when Gateway không gia hạn lease (mất kết nối): Phần tử lease hết hạn bị zremrangebyscore xoá, zcard = 0 nên hiển thị offline.

## Endpoints covered

- GET /users/:id/presence [module:user] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/user.md
