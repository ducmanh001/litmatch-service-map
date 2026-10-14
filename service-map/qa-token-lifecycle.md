# Vòng đời token và phiên: access, refresh, CSRF, cookie?

id: qa:token-lifecycle
category: authn
services: module:auth

Access token ngắn hạn kèm refresh token xoay vòng: mỗi lần refresh phát token mới, dùng lại token cũ bị coi là bị lộ và thu hồi cả họ token. Refresh token nằm trong cookie httpOnly nên cần thêm CSRF double-submit.

### Xoay vòng và phát hiện tái sử dụng

Mỗi refresh token chỉ dùng một lần; client gửi hai refresh song song thì request thua bị coi là tái sử dụng và cả family bị thu hồi, nên client phải single-flight.

services: auth; evidence: apps/core-api/src/modules/auth/auth.controller.ts:113

### Cookie httpOnly

Refresh token nằm trong cookie httpOnly/SameSite nên JavaScript không đọc được, chặn XSS đánh cắp token (docs/adr/0007).

services: auth; evidence: apps/core-api/src/modules/auth/auth.controller.ts:113

### CSRF

Vì cookie tự đính kèm nên CsrfGuard bắt header CSRF khớp token double-submit trên các route dùng cookie.

services: auth; evidence: apps/core-api/src/common/csrf/csrf-token.ts:6

## Alternatives not chosen

- Lưu refresh token ở localStorage: Dễ dùng nhưng JavaScript đọc được nên XSS lấy được; repo đã chọn cookie httpOnly (docs/adr/0007).
