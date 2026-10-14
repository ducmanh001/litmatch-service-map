# Nâng cấp tài khoản khách thành tài khoản thật

id: flow:auth-guest-upgrade
group: Danh tính
services: module:auth, module:user

Người dùng đang là guest gắn danh tính OTP điện thoại hoặc social (Google/Apple/Facebook) vào chính userId hiện tại. Module auth sở hữu danh tính và phiên, module user sở hữu hồ sơ users (cờ isGuest), nên việc gắn identity, bỏ identity guest, đổi isGuest và thu hồi refresh session phải nằm chung một transaction.

Hai endpoint /auth/upgrade/otp và /auth/upgrade/social cùng dùng linkIdentityAndUpgrade: xác minh, gắn identity + đổi isGuest + revoke phiên cũ trong một transaction, rồi mới cấp phiên mới.

## Participants

- web (client): Web client
- auth (service): Auth
- user (service): User
- postgres (infra): PostgreSQL
- external:social (external): Google/Apple/Facebook

## Steps

1. web -> auth [call/http]: POST /auth/upgrade/otp {phone, code} bằng access token của guest (Route yêu cầu JWT (APP_GUARD) và bị throttle 10 request/phút.)
2. auth -> postgres [call/shared-db]: Kiểm tra OTP bật, identity phone đã thuộc user này chưa (auth_identities) (Nếu identity đã thuộc user khác thì trả 409 IDENTITY_ALREADY_LINKED; nếu đã thuộc chính user này thì bỏ qua bước verify OTP.) {apps/core-api/src/modules/auth/auth.service.ts:205}
3. auth -> postgres [call/shared-db]: verifyOtp: đếm lần thử, so mã HMAC, consume OTP (phone_otps) {apps/core-api/src/modules/auth/services/otp.service.ts:119}
4. auth -> postgres [call/shared-db]: Mở transaction, khoá dòng users của guest (pessimistic_write) {apps/core-api/src/modules/auth/auth.service.ts:225}
5. auth -> user [call/di-call]: Nếu không thấy dòng user thì UserService.getByIdOrThrow {apps/core-api/src/modules/auth/auth.service.ts:229}
6. auth -> postgres [call/shared-db]: Chèn identity (nếu chưa có), nếu còn guest thì xoá identity guest, isGuest=false (Kiểm tra user active, identity không thuộc user khác, và user còn là guest (nếu không là 409 GUEST_UPGRADE_NOT_ALLOWED). Auth ghi thẳng entity User bằng manager của transaction.) {apps/core-api/src/modules/auth/auth.service.ts:232}
7. auth -> postgres [call/shared-db]: Nếu user còn guest: revoke mọi refresh session cũ cùng manager rồi commit {apps/core-api/src/modules/auth/auth.service.ts:266}
8. auth -> postgres [call/shared-db]: Sau commit: ký access token mới (isGuest=false), lưu hash refresh token {apps/core-api/src/modules/auth/services/token.service.ts:53}
9. auth -> web [return/http]: 200 {accessToken, csrfToken, userId, isGuest:false} + cookie refresh
10. web -> auth [call/http]: POST /auth/upgrade/social {provider, idToken} (Cùng guard JWT và throttle 10 request/phút như nhánh OTP.)
11. auth -> external:social [call/http]: Xác minh idToken (JWKS Google/Apple, hoặc debug_token của Facebook) (Lỗi xác minh trả 401 SOCIAL_TOKEN_INVALID với thông báo chung.) {apps/core-api/src/modules/auth/services/social-verifier.ts:72}
12. auth -> postgres [call/shared-db]: linkIdentityAndUpgrade: cùng transaction gắn identity, isGuest=false, revoke phiên {apps/core-api/src/modules/auth/auth.service.ts:224}
13. auth -> postgres [call/shared-db]: Sau commit: ký access token mới và lưu refresh session mới {apps/core-api/src/modules/auth/services/token.service.ts:53}
14. auth -> web [return/http]: 200 {accessToken, csrfToken, userId, isGuest:false} + cookie refresh

## Invariants

- Thêm identity, xoá identity guest, đổi isGuest=false và revoke refresh session phải cùng một transaction (cùng manager) để không commit nâng cấp nửa chừng: credential guest và phiên cũ không được đi theo thành quyền của tài khoản thật.
- Phiên mới chỉ được cấp (issue) sau khi transaction nâng cấp xong, vì access token mang isGuest và refresh session cũ vừa bị revoke trong transaction; cấp trước sẽ bị revoke ngay.
- Dòng users bị khoá pessimistic_write trước khi đọc identity, nên hai request nâng cấp song song không cùng đổi trạng thái; unique (provider, providerUid) là chốt cuối khi tranh chấp.

## Failure modes

- when Identity phone/social đã gắn với user khác: 409 IDENTITY_ALREADY_LINKED, không tự động merge tài khoản, transaction không commit.
- when User không còn là guest và identity chưa có: 409 GUEST_UPGRADE_NOT_ALLOWED, không gắn identity mới qua luồng upgrade.
- when OTP hết hạn, sai hoặc vượt số lần thử: Báo lỗi OTP_EXPIRED/OTP_INVALID/OTP_TOO_MANY_ATTEMPTS trước khi mở transaction, không đổi isGuest.
- when Vi phạm unique do request song song cùng identity: Đọc lại identity thắng; nếu thuộc chính user thì update isGuest có điều kiện (chỉ khi ghi được dòng mới revoke phiên) rồi đăng nhập tiếp, nếu không thì 409.
- when Token social không hợp lệ hoặc provider chưa cấu hình: 401 SOCIAL_TOKEN_INVALID hoặc 400 SOCIAL_PROVIDER_NOT_SUPPORTED, không đụng DB.

## Endpoints covered

- POST /auth/upgrade/otp [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
- POST /auth/upgrade/social [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
