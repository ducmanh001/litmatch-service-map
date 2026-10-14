# Techniques used by app-shell

id: techniques:app-shell
module: module:app-shell

### Guard toàn cục + @Public() để mở (global-guard-public-optout)

- how: Guard JWT đăng ký toàn cục; route nào muốn mở phải gắn @Public() tường minh.
- why: An toàn theo mặc định: quên cấu hình thì bị chặn chứ không bị mở.
- applied here: HealthController và CapabilitiesController gắn @Public() cùng @SkipThrottle() để probe và capabilities không cần JWT.
- endpoints: e1, e2, e3, e4
- alternative: Gắn guard từng controller: dễ sót route.

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: CapabilitiesDto không lưu trữ: mỗi lần gọi tính lại từ env thật (credential, NODE_ENV) cộng danh sách CAPABILITY_MAINTENANCE_FEATURES, nên trạng thái luôn khớp cấu hình đang chạy.
- endpoints: e4
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Chỉ GET /health/ready có @Throttle 30 lần/phút; /health, /health/live và /capabilities (e4) gắn @SkipThrottle nên không chịu ThrottlerGuard.
- endpoints: e3
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: ReadinessService coi lỗi hoặc timeout 1s của SELECT 1 / PING Redis là 'down', trả 503 để orchestrator rút pod thay vì giả định dependency còn sống.
- endpoints: e3
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Các route health và capabilities có @ApiOperation mô tả hợp đồng, capabilities là contract runtime cho mọi frontend.
- endpoints: e1, e2, e3, e4
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: Endpoint public chỉ trả clientId (không bí mật) và chỉ khi provider available; secret như AUTH_FACEBOOK_APP_SECRET, PAYOS_API_KEY chỉ dùng để tính cờ, không đưa vào response.
- endpoints: e4
- alternative: 

### Liveness / readiness probe (health-readiness)

- how: Liveness và readiness probe cho orchestrator.
- why: K8s biết khi nào gửi traffic hoặc khởi động lại.
- applied here: HealthController tách live, ready dùng ReadinessService (Postgres và Redis) cho liveness/readiness probe.
- endpoints: e1, e2, e3
- alternative: 

