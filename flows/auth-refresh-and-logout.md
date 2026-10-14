# Làm mới phiên và đăng xuất (cookie httpOnly + CSRF)

id: flow:auth-refresh-and-logout
group: Danh tính
services: module:auth, module:user

Web giữ refresh token trong cookie httpOnly; khi access token hết hạn web gọi /auth/refresh để xoay vòng token, và /auth/logout để thu hồi phiên. Module auth giữ phiên refresh nhưng phải hỏi module user để biết user còn hợp lệ (không bị ban) tại thời điểm làm mới.

Hình cho thấy auth xoay vòng refresh token trong transaction, kiểm tra lại user bên module user, rồi đăng xuất thu hồi token.

## Participants

- web (client): Web client
- auth (service): auth
- user (service): user
- postgres (infra): PostgreSQL

## Steps

1. web -> auth [call/http]: POST /auth/refresh kèm cookie refresh và header X-CSRF-Token (Route có @UseGuards(CsrfGuard): cookie CSRF phải khớp header, sai thì trả 401 CSRF_TOKEN_INVALID trước khi vào service.)
2. auth -> postgres [call/shared-db]: Tìm phiên theo SHA-256 của refresh token; thiếu/đã thu hồi/hết hạn thì từ chối {apps/core-api/src/modules/auth/services/token.service.ts:71}
3. auth -> user [call/di-call]: Lấy user theo id để biết isGuest và role dùng ký access token {apps/core-api/src/modules/auth/services/token.service.ts:81}
4. auth -> postgres [call/shared-db]: Khoá dòng phiên cũ, đánh dấu rotated, chèn refresh token mới cùng family (Nếu phiên đã rotated hoặc UPDATE có điều kiện không ghi được dòng nào thì thu hồi cả family và trả 'reused'.) {apps/core-api/src/modules/auth/repositories/typeorm-refresh-session.adapter.ts:39}
5. postgres -> auth [return/shared-db]: Kết quả rotated, invalid hoặc reused (reused trả 401 REFRESH_TOKEN_REUSED)
6. auth -> user [call/di-call]: Đọc lại user để kiểm tra trạng thái đúng thời điểm làm mới {apps/core-api/src/modules/auth/auth.service.ts:128}
7. auth -> postgres [call/shared-db]: Nếu user không active thì thu hồi refresh token vừa cấp rồi ném lỗi (Nhánh lỗi của assertActive: chỉ chạy khi user bị chặn giữa hai lần refresh.) {apps/core-api/src/modules/auth/auth.service.ts:132}
8. auth -> web [return/http]: 200: accessToken + csrfToken mới; Set-Cookie refresh và CSRF mới (csrfToken sinh bằng generateCsrfToken(); refreshToken được bóc khỏi body và chỉ nằm trong cookie httpOnly.)
9. web -> auth [call/http]: POST /auth/logout kèm cookie refresh và X-CSRF-Token (Cùng CsrfGuard như refresh.)
10. auth -> postgres [call/shared-db]: Đặt revokedAt cho phiên chưa thu hồi khớp hash (idempotent) {apps/core-api/src/modules/auth/services/token.service.ts:107}
11. auth -> web [return/http]: 204 No Content và xoá các cookie xác thực {apps/core-api/src/modules/auth/auth.controller.ts:140}

## Invariants

- Đánh dấu rotated phải là UPDATE có điều kiện rotated_at IS NULL trong transaction có khoá pessimistic_write, nên hai request song song cùng một token chỉ một request thắng; request thua bị coi là reuse và thu hồi cả family.
- auth phải đọc lại user ở module user sau khi rotate để user bị ban giữa hai lần refresh không giữ được phiên mới; khi đó token vừa cấp bị thu hồi.
- Cả refresh và logout cần CsrfGuard (double-submit cookie và header) vì refresh token nằm trong cookie httpOnly được trình duyệt tự gửi.

## Failure modes

- when Thiếu hoặc sai CSRF token: CsrfGuard ném 401 CSRF_TOKEN_INVALID, không chạm DB.
- when Thiếu cookie refresh, hoặc refresh token không tồn tại, đã thu hồi hay hết hạn: Trả 401 REFRESH_TOKEN_INVALID.
- when Refresh token đã được xoay vòng dùng lại: Thu hồi cả family trong transaction và trả 401 REFRESH_TOKEN_REUSED.
- when User bị chặn giữa hai lần refresh: assertActive ném lỗi, token mới vừa cấp bị thu hồi, không có cookie mới.

## Endpoints covered

- POST /auth/refresh [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
- POST /auth/logout [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
