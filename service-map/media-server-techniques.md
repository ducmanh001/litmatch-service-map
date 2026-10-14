# Techniques used by media-server

id: techniques:media-server
module: module:media-server

### Mạng hostNetwork cho RTC (other:host-network-rtc)

- how: Deployment k8s dùng hostNetwork: true để client tới thẳng 7881 và dải UDP 50000-50200 trên IP Node (ADR 0005); đánh đổi mất cô lập mạng, tối đa 1 pod mỗi Node.
- why: Deployment k8s dùng hostNetwork: true để client tới thẳng 7881 và dải UDP 50000-50200 trên IP Node (ADR 0005); đánh đổi mất cô lập mạng, tối đa 1 pod mỗi Node.
- applied here: Deployment k8s dùng hostNetwork: true để client tới thẳng 7881 và dải UDP 50000-50200 trên IP Node (ADR 0005); đánh đổi mất cô lập mạng, tối đa 1 pod mỗi Node.
- endpoints: e51, e52
- alternative: 

### Xác thực chữ ký/token của webhook (webhook-signature)

- how: Webhook phải được xác minh chữ ký/token của nhà cung cấp trên body gốc trước khi xử lý.
- why: Endpoint webhook công khai nên chữ ký là cách duy nhất chứng minh nguồn gửi.
- applied here: LiveKit ký webhook bằng JWT api_key (LIVEKIT_WEBHOOK_API_KEY) và controller core-api verify trên rawBody trước khi tin.
- endpoints: e54, e55
- alternative: 

### Capability token ngắn hạn (grant của LiveKit) (capability-token)

- how: Cấp token ngắn hạn mang sẵn quyền cụ thể (publish, subscribe) cho đúng phòng.
- why: Quyền được enforce ngay ở hệ thống media chứ không tin client.
- applied here: SFU chỉ nhận client có JWT do core-api mint, TTL ngắn, grant canPublish theo role và canPublishData: true; secret chỉ ở core-api và LiveKit.
- endpoints: e50
- alternative: 

### Cô lập secret (chỉ core-api giữ khoá) (secret-isolation)

- how: Khoá và secret chỉ nằm ở core-api, không ra client.
- why: Giảm bề mặt lộ khoá: khoá ký và secret chỉ nằm ở backend, phía client không bao giờ nhận được.
- applied here: Dev: key/secret inject bằng LIVEKIT_KEYS từ .env, không nằm trong livekit.yaml. K8s: key/secret buộc nằm trong ConfigMap media-server-config (placeholder REPLACE_ME_KEY), phải sinh lúc deploy từ secret manager, không commit.
- endpoints: e50
- alternative: 

### Liveness / readiness probe (health-readiness)

- how: Liveness và readiness probe cho orchestrator.
- why: K8s biết khi nào gửi traffic hoặc khởi động lại.
- applied here: Deployment k8s dùng HTTP GET path gốc cổng 7880 làm probe liveness/readiness vì LiveKit không có /health/live riêng; response chi tiết chưa kiểm chứng.
- endpoints: e50
- alternative: 

### Metrics Prometheus (prometheus-metrics)

- how: Xuất metrics cho Prometheus.
- why: Có số liệu theo thời gian để đo SLO và để rule cảnh báo tự kích hoạt khi hệ thống lệch.
- applied here: LiveKit mở prometheus_port 6789 và deployment gắn annotation prometheus.io/scrape, port, path /metrics để Prometheus scrape.
- endpoints: e53
- alternative: 

