# Đăng nhập bằng OTP điện thoại

id: flow:auth-otp-login
group: Danh tính
services: module:auth, module:user

Người dùng xin mã OTP rồi xác minh để nhận phiên đăng nhập. Auth giữ danh tính (auth_identities), OTP và refresh session, còn hồ sơ người dùng do User sở hữu nên Auth phải gọi UserService để đọc hoặc tạo user.

Hai endpoint OTP: xin mã (Auth ghi phone_otp), rồi xác minh, tìm hoặc tạo user qua User và cấp token.

## Participants

- web (client): Web client
- auth (service): Auth
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> auth [call/http]: POST /auth/otp/request với số điện thoại (throttle 5/phút theo IP) (Controller có @Throttle limit 5 mỗi phút theo IP; giới hạn theo số điện thoại nằm trong OtpService.) {apps/core-api/src/modules/auth/auth.controller.ts:64}
2. auth -> postgres [call/shared-db]: Đếm OTP trong 1 giờ, vô hiệu mã cũ, lưu hash mã mới vào phone_otp (Không gửi SMS: mã được trả thẳng trong response. Quá giới hạn mỗi giờ thì ném OTP_REQUEST_RATE_LIMITED (429).) {apps/core-api/src/modules/auth/services/otp.service.ts:32}
3. auth -> web [return/http]: Trả code và ttlSeconds để client hiển thị và tự điền
4. web -> auth [call/http]: POST /auth/otp/verify với phone và code (throttle 10/phút) {apps/core-api/src/modules/auth/auth.controller.ts:75}
5. auth -> postgres [call/shared-db]: Đọc OTP mới nhất, tăng attempt_count có điều kiện, so hash (UPDATE có điều kiện attempt_count < max nên request song song không vượt giới hạn; sai mã ném OTP_INVALID.) {apps/core-api/src/modules/auth/services/otp.service.ts:74}
6. auth -> postgres [call/shared-db]: Đánh dấu consumed_at (UPDATE có điều kiện consumed_at IS NULL) (Một mã chỉ dùng được đúng một lần, request song song thua sẽ bị OTP_INVALID.) {apps/core-api/src/modules/auth/services/otp.service.ts:119}
7. auth -> postgres [call/shared-db]: Tìm auth_identities theo (phone, số điện thoại) {apps/core-api/src/modules/auth/auth.service.ts:156}
8. auth -> user [call/di-call]: Đã có identity: getByIdOrThrow rồi kiểm tra user còn Active (Nhánh user cũ: assertActive ném USER_BANNED (403) nếu tài khoản bị khóa.) {apps/core-api/src/modules/auth/auth.service.ts:162}
9. auth -> user [call/di-call]: Chưa có identity: createWithManager tạo user mới trong transaction (Nhánh user mới: bảng users thuộc User nên Auth phải nhờ UserService tạo bằng cùng manager.) {apps/core-api/src/modules/auth/auth.service.ts:170}
10. auth -> postgres [call/shared-db]: Lưu AuthIdentity (phone) cùng transaction với user mới (Unique (provider, providerUid) là chốt cuối khi hai request đầu tiên chạy song song; bên thua đọc lại identity của bên kia.) {apps/core-api/src/modules/auth/auth.service.ts:174}
11. auth -> postgres [call/shared-db]: Ký JWT access token, lưu hash refresh token (family mới) vào refresh session (RefreshSessionPort được nối vào TypeOrmRefreshSessionAdapter trong auth.module.ts.) {apps/core-api/src/modules/auth/services/token.service.ts:53}
12. auth -> web [return/http]: Set cookie httpOnly refresh và CSRF, trả accessToken và userId (finalizeSession bóc refreshToken khỏi body, chỉ đặt vào cookie.)

## Invariants

- Hồ sơ user thuộc module User nên Auth không ghi bảng users trực tiếp mà gọi UserService.createWithManager với cùng EntityManager, để user và identity cùng commit hoặc cùng rollback.
- OTP phải được consume (UPDATE có điều kiện consumed_at IS NULL) trước khi tìm hoặc tạo user, nên một mã không thể đổi lấy hai phiên dù có request song song.
- Token chỉ được cấp sau khi user được xác nhận Active, vì Auth cần trạng thái user từ User ngay tại thời điểm đăng nhập.

## Failure modes

- when Số lần xin OTP của một số điện thoại vượt AUTH_OTP_REQUESTS_PER_HOUR: Auth ném OTP_REQUEST_RATE_LIMITED (429), không tạo mã mới.
- when OTP hết hạn hoặc chưa được yêu cầu, nhập sai mã, hoặc vượt AUTH_OTP_MAX_ATTEMPTS: OTP_EXPIRED (400), OTP_INVALID (400) hoặc OTP_TOO_MANY_ATTEMPTS (429), không tạo phiên.
- when User gắn với identity đã bị khóa: assertActive ném USER_BANNED (403) nên không cấp token; OTP đã bị consume nên phải xin mã mới.
- when Hai request đầu tiên tạo cùng số điện thoại song song: Một bên dính unique violation, đọc lại identity của bên kia và tiếp tục đăng nhập vào cùng user.

## Endpoints covered

- POST /auth/otp/request [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
- POST /auth/otp/verify [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
