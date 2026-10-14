# Kiểm tra sẵn sàng của core-api (readiness)

id: flow:app-shell-health-and-readiness
group: Vận hành (admin)
services: module:app-shell

Probe Docker/k8s gọi /health/ready để biết core-api có nhận traffic được không; app-shell phải hỏi thật PostgreSQL và Redis song song với timeout 1 giây rồi trả 200 hoặc 503.

Probe gọi /health/ready, app-shell chạy song song SELECT 1 trên PostgreSQL và PING trên Redis, rồi trả 200 hoặc 503.

## Participants

- web (client): Probe (Docker/k8s)
- app-shell (service): app-shell
- postgres (infra): PostgreSQL
- redis (infra): Redis

## Steps

1. web -> app-shell [call/http]: GET /health/ready (route public, throttle 30 lượt/phút) (Route đánh dấu @Public nên không cần JWT.)
2. app-shell -> app-shell [call/di-call]: HealthController.ready gọi ReadinessService.check() {apps/core-api/src/app/health.controller.ts:47}
3. app-shell -> postgres [call/shared-db]: SELECT 1 với timeout 1 giây {apps/core-api/src/app/readiness.service.ts:57}
4. app-shell -> redis [call/redis-state]: PING với timeout 1 giây (kết nối lazy nếu chưa có) {apps/core-api/src/app/readiness.service.ts:67}
5. postgres -> app-shell [return/shared-db]: Trả kết quả: 'up'; lỗi hoặc quá hạn thì 'down'
6. redis -> app-shell [return/redis-state]: Trả PONG: 'up'; lỗi hoặc quá hạn thì 'down'
7. app-shell -> app-shell [call/di-call]: Gộp kết quả: cả hai 'up' thì status 'ok', ngược lại 'unavailable' {apps/core-api/src/app/readiness.service.ts:44}
8. app-shell -> web [return/http]: Trả 200 {status, checks}; status không phải 'ok' thì 503

## Invariants

- Readiness phải hỏi thật PostgreSQL và Redis (SELECT 1, PING) thay vì chỉ báo process còn sống; liveness /health/live riêng không gọi dependency, nên probe có thể tách 'đang chạy' khỏi 'sẵn sàng nhận traffic'.
- Hai kiểm tra chạy song song bằng Promise.all và mỗi cái có timeout 1 giây, nên một dependency treo không làm probe treo; lỗi được bắt và đổi thành 'down' thay vì ném exception.

## Failure modes

- when PostgreSQL lỗi hoặc không trả lời trong 1 giây: checks.postgres = 'down', status 'unavailable', endpoint trả HTTP 503
- when Redis không kết nối được hoặc PING quá 1 giây: checks.redis = 'down', status 'unavailable', endpoint trả HTTP 503; lỗi event của Redis bị nuốt, không gây unhandled event
- when Probe gọi quá 30 lượt/phút: Throttler áp giới hạn cho route này (khai báo @Throttle limit 30, ttl 1 phút)

## Endpoints covered

- GET /health/ready [module:app-shell] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/app-shell.md
