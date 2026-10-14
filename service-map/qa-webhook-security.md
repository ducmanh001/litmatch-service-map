# Bảo mật webhook từ bên ngoài (thanh toán, LiveKit)?

id: qa:webhook-security
category: security
services: module:economy, module:calling, module:party-room, module:media-server

Mọi webhook công khai đều tự xác minh nguồn gửi trước khi làm gì: PayOS bằng HMAC-SHA256, Apple bằng JWS (kiểm chuỗi chứng chỉ x5c lên Apple Root CA), Google RTDN bằng OIDC token (kiểm issuer, audience, service account), LiveKit bằng chữ ký khoá chung (receiveWebhook).

### PayOS

Tính lại HMAC-SHA256 bằng checksum key rồi mới ghi ledger; idempotency key payos-credit:{orderCode} chặn cộng diamond lần hai khi payOS gửi lại.

services: economy; evidence: apps/core-api/src/modules/economy/dto/payos-webhook.dto.ts:11

### Apple và Google

Apple gửi signedPayload JWS, server kiểm chuỗi x5c và bundleId; Google gửi Pub/Sub push kèm Bearer OIDC, server kiểm issuer, audience và email service account. Endpoint là @Public nên chữ ký là lớp xác thực duy nhất.

services: economy; evidence: apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:129

### LiveKit

Webhook phòng và cuộc gọi được xác minh bằng receiveWebhook với cùng cặp khoá API; không hợp lệ thì trả 401 nên call không bao giờ chuyển sang active.

services: calling, party-room, media-server; evidence: apps/core-api/src/modules/calling/webhooks/livekit-webhook.controller.ts:51, apps/core-api/src/modules/party-room/webhooks/party-livekit-webhook.controller.ts:51

## Alternatives not chosen

- Tin returnUrl hoặc redirect từ trình duyệt: Trình duyệt có thể bị giả mạo; chỉ chữ ký server-to-server mới là bằng chứng đã trả tiền.

## Caveats

- Google RTDN chỉ được log (CANCELED không đồng nghĩa voided), hoàn tiền Google dựa vào job quét Voided Purchases API.
