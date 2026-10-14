# Đăng nhập khách rồi mở kết nối realtime

id: flow:auth-guest-login-and-socket
group: Danh tính
services: module:auth, module:user, module:signaling-gateway

Người dùng mới lấy phiên khách từ core-api (auth + user + PostgreSQL), rồi mở Socket.IO tới signaling-gateway bằng chính access token đó; gateway tự verify token, giữ quota kết nối và presence trong Redis.

Hai chặng nối tiếp: core-api cấp access token khách, sau đó signaling-gateway verify token đó và giữ lease kết nối trong Redis.

## Participants

- web (client): Web client
- auth (service): auth (core-api)
- user (service): user (core-api)
- postgres (infra): PostgreSQL
- signaling-gateway (service): signaling-gateway
- redis (infra): Redis

## Steps

1. web -> auth [call/http]: POST /auth/guest với deviceId
2. auth -> postgres [call/shared-db]: Tìm AuthIdentity của thiết bị khách {apps/core-api/src/modules/auth/auth.service.ts:156}
3. auth -> user [call/di-call]: Nếu identity đã có: lấy user hiện tại (Cũng gọi assertActive: user bị khoá thì ném USER_BANNED (403); identity guest mà user đã nâng cấp (isGuest=false) thì ném GUEST_IDENTITY_RETIRED (401).) {apps/core-api/src/modules/auth/auth.service.ts:162}
4. auth -> user [call/di-call]: Nếu chưa có: tạo user khách trong transaction {apps/core-api/src/modules/auth/auth.service.ts:170}
5. auth -> postgres [call/shared-db]: Lưu AuthIdentity cùng transaction {apps/core-api/src/modules/auth/auth.service.ts:174}
6. auth -> postgres [call/shared-db]: Nhánh race: unique violation thì đọc lại identity của request thắng (Chỉ chạy trong catch khi transaction tạo user + identity dính unique violation (hai request song song cùng deviceId); luồng thường không đọc lại sau commit.) {apps/core-api/src/modules/auth/auth.service.ts:185}
7. auth -> user [call/di-call]: Nhánh race: lấy user của identity thắng rồi kiểm tra active/guest {apps/core-api/src/modules/auth/auth.service.ts:192}
8. auth -> auth [call/di-call]: Ký access JWT (sub = userId, kèm isGuest, role, jti) (TokenService.issueForUser ký access JWT bằng jwtService.signAsync; refresh token ngẫu nhiên được hash rồi lưu ở bước kế tiếp.) {apps/core-api/src/modules/auth/services/token.service.ts:47}
9. auth -> postgres [call/shared-db]: Lưu hash refresh token (RefreshSessionPort.issue) vào refresh_tokens {apps/core-api/src/modules/auth/repositories/typeorm-refresh-session.adapter.ts:18}
10. auth -> web [return/http]: Trả accessToken, csrfToken, guestDeviceToken; refresh token đi qua cookie (finalizeSession set cookie (refresh token httpOnly, csrf) qua setAuthCookies; body JSON gồm accessToken, csrfToken, expiresIn, userId, isGuest, guestDeviceToken, không chứa refreshToken. guestDeviceToken do GuestDeviceTokenService.issue tạo sau khi đăng nhập.)
11. web -> signaling-gateway [call/socket]: Handshake Socket.IO kèm handshake.auth.token {apps/signaling-gateway/src/app/signaling.gateway.ts:209}
12. signaling-gateway -> signaling-gateway [call/di-call]: Tự verify JWT, gán userId = payload.sub {apps/signaling-gateway/src/app/signaling.gateway.ts:214}
13. signaling-gateway -> redis [call/redis-state]: Acquire lease quota và ghi presence (script Lua) {apps/signaling-gateway/src/app/connection-quota.service.ts:126}
14. redis -> signaling-gateway [return/redis-state]: Kết quả 1 = nhận kết nối, 0 = vượt giới hạn
15. signaling-gateway -> web [return/socket]: Kết nối được nhận, socket vào room user:{userId} {apps/signaling-gateway/src/app/signaling.gateway.ts:173}
16. signaling-gateway -> redis [async/redis-state]: Định kỳ gia hạn lease (leaseMs/3), thất bại thì ngắt socket {apps/signaling-gateway/src/app/signaling.gateway.ts:290}

## Invariants

- Access token phải được auth ký xong trước khi mở socket vì gateway chỉ chấp nhận handshake có handshake.auth.token verify được bằng JwtService; không có token hợp lệ thì handshake bị từ chối trước khi thành socket.
- Tạo user và AuthIdentity nằm cùng một dataSource.transaction (manager.save, createWithManager) để không sinh user khách mà thiếu identity.
- Gateway lấy userId từ payload.sub của token, không từ client, rồi tự join socket vào room user:{userId}; client không chọn room.
- Lease quota được acquire trong middleware handshake, trước khi socket được tạo; quota và presence cập nhật chung trong một script Lua trên Redis.

## Failure modes

- when Token thiếu, rỗng, sai chữ ký hoặc thiếu sub: authenticate ném lỗi Unauthorized, handshake bị từ chối
- when Acquire lease trả 0 (vượt số kết nối cho phép): admitConnection trả false, middleware ném CONNECTION_LIMIT_ERROR
- when Redis lỗi lúc acquire: Gateway thử release lease, báo Sentry và từ chối bằng CONNECTION_QUOTA_UNAVAILABLE_ERROR
- when Refresh lease trả false hoặc lỗi: Gateway ngắt socket (fail closed)

## Endpoints covered

- POST /auth/guest [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
