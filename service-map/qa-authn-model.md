# Xác thực người dùng bằng mô hình nào?

id: qa:authn-model
category: authn
services: module:auth, module:user, module:signaling-gateway

JWT bearer không trạng thái, kiểm bằng JwtAuthGuard đăng ký toàn cục: route nào muốn mở phải gắn @Public(). Có ba đường vào (khách kèm token thiết bị, OTP điện thoại, ID token Apple/Google/Facebook) và gateway WebSocket xác minh đúng token đó bằng chung khoá.

### Guard toàn cục

ThrottlerGuard, JwtAuthGuard rồi RolesGuard được đăng ký làm APP_GUARD theo đúng thứ tự: chặn spam trước, xác thực sau, kiểm vai trò cuối. Quên cấu hình thì route bị chặn chứ không bị mở.

services: auth, user; evidence: apps/core-api/src/app/app.module.ts:154

### Ký và xác minh token

core-api ký access token (payload gồm sub, isGuest, role, jti) bằng JWT_SECRET và tự xác minh ở mọi request; không có kho phiên phía server.

services: auth; evidence: apps/core-api/src/modules/auth/services/token.service.ts:27

### Ba đường đăng nhập

Khách nhận token thiết bị ký HMAC để tính hạn mức; OTP điện thoại chỉ lưu dạng băm HMAC kèm giới hạn thử; Apple, Google, Facebook thì server tự xác minh ID token với nhà cung cấp.

services: auth; evidence: apps/core-api/src/modules/auth/services/otp.service.ts:15, apps/core-api/src/modules/auth/auth.module.ts:41, apps/core-api/src/modules/auth/services/guest-device-token.service.ts:11

### WebSocket

signaling-gateway xác minh cùng access token lúc handshake bằng chung JWT_SECRET; gateway không bao giờ ký token và không gọi core-api để hỏi. [10/10] Có thể xoay khoá không đăng xuất mọi người nhờ JWT_SECRET_PREVIOUS (chỉ thử khi chữ ký không khớp).

services: signaling-gateway, auth; evidence: apps/signaling-gateway/src/app/signaling.gateway.ts:157

## Alternatives not chosen

- Session phía server (cookie + kho phiên): Suy luận: cần kho phiên dùng chung và gateway tách riêng phải hỏi lại core-api; JWT cho hai deployable chỉ cần chung khoá để xác minh.
- IdP/OAuth server riêng (Keycloak, Auth0): Suy luận: thêm một thành phần vận hành; hệ thống chỉ cần danh tính nội bộ và đã tự xác minh ID token của bên thứ ba.

## Caveats

- Access token không thu hồi tức thì: trạng thái bị ban chỉ được kiểm lại ở refresh/đăng nhập, nên thao tác nhạy cảm tự kiểm lại (ví dụ soul-match kiểm UserStatus.Banned lúc gửi tin).
