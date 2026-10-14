# Đăng nhập bằng Apple, Google hoặc Facebook

id: flow:auth-social-login
group: Danh tính
services: module:auth, module:user

Client gửi ID token (Google/Apple) hoặc access token (Facebook) lên POST /auth/social. Server không tin token mà tự xác minh với nhà cung cấp, rồi module auth tìm hoặc tạo danh tính, nhờ module user tạo hồ sơ, và cuối cùng ký access token cùng refresh token.

Luồng POST /auth/social: xác minh token với nhà cung cấp, tìm hoặc tạo user trong một transaction, rồi cấp phiên.

## Participants

- web (client): Web client
- auth (service): Auth
- external:social (external): Apple/Google/Facebook
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> auth [call/http]: POST /auth/social gửi provider và idToken (Route @Public (không cần JWT) và bị throttle 10 request/phút.) {apps/core-api/src/modules/auth/auth.controller.ts:90}
2. auth -> external:social [call/sdk]: Google/Apple: jwtVerify chữ ký, issuer, audience qua JWKS (Facebook đi nhánh khác: gọi Graph debug_token bằng App Secret (bước kế tiếp là nhánh thay thế).) {apps/core-api/src/modules/auth/services/social-verifier.ts:37}
3. auth -> external:social [call/http]: Facebook: fetch debug_token, kiểm is_valid và app_id (Timeout 5 giây; token sai hoặc lỗi mạng đều trả 401 SOCIAL_TOKEN_INVALID với thông báo chung.) {apps/core-api/src/modules/auth/services/social-verifier.ts:90}
4. auth -> postgres [call/shared-db]: Tìm auth_identities theo (provider, providerUid) {apps/core-api/src/modules/auth/auth.service.ts:156}
5. auth -> user [call/di-call]: Danh tính đã có: lấy user, kiểm status Active (assertActive ném 403 USER_BANNED nếu tài khoản bị khoá.) {apps/core-api/src/modules/auth/auth.service.ts:162}
6. auth -> user [call/di-call]: Chưa có danh tính: tạo user trong transaction (Nickname dạng User-<số ngẫu nhiên>, isGuest = false.) {apps/core-api/src/modules/auth/auth.service.ts:170}
7. auth -> postgres [call/shared-db]: Lưu AuthIdentity cùng transaction với user {apps/core-api/src/modules/auth/auth.service.ts:174}
8. auth -> postgres [call/shared-db]: Race unique violation: đọc lại identity của request thắng (Chỉ chạy khi transaction lỗi unique violation; sau đó lấy user qua UserService và kiểm assertActive.) {apps/core-api/src/modules/auth/auth.service.ts:185}
9. auth -> auth [call/sdk]: Ký JWT access token (sub, isGuest, role, jti) {apps/core-api/src/modules/auth/services/token.service.ts:47}
10. auth -> postgres [call/shared-db]: Lưu băm refresh token (tokenHash, familyId, expiresAt) {apps/core-api/src/modules/auth/repositories/typeorm-refresh-session.adapter.ts:18}
11. auth -> web [return/http]: 200: accessToken, csrfToken; refreshToken đặt vào cookie httpOnly (finalizeSession gọi setAuthCookies rồi bóc refreshToken khỏi body trả về.)

## Invariants

- Server phải tự xác minh token với nhà cung cấp trước khi đụng DB, vì không được tin token do client gửi lên mà chưa kiểm chữ ký, issuer và audience.
- User và AuthIdentity được tạo trong cùng một dataSource.transaction để không có user mồ côi hoặc identity trỏ vào user không tồn tại; auth sở hữu danh tính còn user sở hữu hồ sơ.
- Unique (provider, providerUid) ở DB là chốt chặn cuối khi hai request đầu tiên chạy song song: bên thua đọc lại identity của bên thắng thay vì tạo user thứ hai.

## Failure modes

- when Token Google/Apple/Facebook sai, hết hạn hoặc lỗi mạng/JWKS: Ném 401 SOCIAL_TOKEN_INVALID với thông báo chung, không tạo user và không cấp phiên; lỗi gốc chỉ được log warn.
- when Provider chưa cấu hình client id hoặc app secret: Ném 400 SOCIAL_PROVIDER_NOT_SUPPORTED.
- when User tương ứng có status khác Active: assertActive ném 403 USER_BANNED, không ký token.
- when Transaction tạo user/identity lỗi: Cả user lẫn identity bị rollback; nếu là unique violation thì đọc lại identity đã có, lỗi khác được ném tiếp.

## Endpoints covered

- POST /auth/social [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
