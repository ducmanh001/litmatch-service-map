# Techniques used by signaling-gateway

id: techniques:signaling-gateway
module: module:signaling-gateway

### Xác thực JWT lúc handshake WebSocket (ws-handshake-auth)

- how: Xác minh JWT ngay lúc bắt tay WebSocket; không hợp lệ thì từ chối kết nối.
- why: Không tồn tại socket vô danh; room lấy từ sub đã xác minh chứ không từ client.
- applied here: SignalingGateway chạy middleware server.use verify handshake.auth.token bằng authenticate() trước khi thành socket; handleConnection chỉ xử lý socket đã có userId.
- endpoints: e50, e51
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Gateway inject JwtService và verifyAsync access token (JWT_SECRET dùng chung với core-api, chỉ verify, không ký), lấy userId từ payload.sub.
- endpoints: e50
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Room user:{userId} do server gán từ payload.sub của JWT đã verify; client không gửi room/event nào ngoài ping, nên không thể nghe event của user khác.
- endpoints: e50, e52
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: Presence không lưu cờ online: isOnline suy từ ZSET realtime:presence:{userId} do lease ghi, xoá member hết hạn rồi zcard > 0; lỗi Redis trả false (fail closed).
- endpoints: e50
- alternative: 

### Socket.IO room theo user (socketio-rooms)

- how: Mỗi socket join room theo user id đã xác thực; tin được phát tới room.
- why: Gửi đúng người nhận và nhiều thiết bị cùng nhận.
- applied here: Namespace /signaling: handleConnection join socket vào room user:{userId}; relay() emit envelope.event/data vào đúng room, channel lạ hoặc JSON hỏng bị bỏ qua.
- endpoints: e50, e52
- alternative: 

### Redis pub/sub fanout (at-most-once, không replay) (redis-pubsub-fanout)

- how: core-api PUBLISH lên kênh realtime:user:{id}; gateway PSUBSCRIBE rồi đẩy xuống socket; không lưu, không replay.
- why: Sự kiện chỉ là gợi ý còn REST là nguồn sự thật nên at-most-once là đủ và rất rẻ.
- applied here: Gateway subscribe pattern REALTIME_USER_CHANNEL_PATTERN từ Redis và relay envelope sang socket; tin mất thì client đọc lại bằng REST.
- endpoints: e52
- alternative: Redis Streams hoặc Kafka bền và replay được nhưng cần consumer group + ack: thừa với tin gợi ý.

### Lease có TTL trong Redis (quota, presence) (redis-ttl-lease)

- how: Khoá hoặc quota có TTL, phải gia hạn định kỳ; chết thì tự hết hạn.
- why: Không cần dọn thủ công khi tiến trình chết.
- applied here: Mỗi socket có lease leaseId TTL WS_CONNECTION_LEASE_MS trong ZSET; setInterval refresh mỗi leaseMs/3, refresh thất bại thì ngắt socket, TTL giải phóng slot khi pod chết.
- endpoints: e50
- alternative: 

### Lua script nguyên tử trong Redis (redis-lua-atomic)

- how: Script Lua chạy nguyên tử trong Redis: nhiều lệnh không bị xen giữa.
- why: Rate limit và lease cần đọc-so sánh-ghi không race.
- applied here: ACQUIRE_SCRIPT chạy redis.eval với 2 key (quota và presence): xoá lease hết hạn theo redis TIME, nếu ZCARD đã bằng WS_MAX_CONNECTIONS_PER_USER thì trả 0, nên đếm đúng xuyên replica.
- endpoints: e50
- alternative: 

### Socket.IO Redis adapter (nhiều instance) (socketio-redis-adapter)

- how: Adapter Redis đồng bộ việc phát tin giữa nhiều instance gateway.
- why: Scale ngang gateway theo số kết nối.
- applied here: main.ts gắn createAdapter(pubClient, subClient) vào CorsIoAdapter trước listen; server.to(room).emit tới được socket ở pod khác, readiness /health/ready đòi adapter ready.
- endpoints: e52
- alternative: 

### Lease có TTL trong Redis (quota, presence) (redis-ttl-lease)

- how: Khoá hoặc quota có TTL, phải gia hạn định kỳ; chết thì tự hết hạn.
- why: Không cần dọn thủ công khi tiến trình chết.
- applied here: ConnectionQuotaService giữ lease với leaseMs trong Redis, gateway refresh định kỳ theo leaseId và release khi đóng; TTL là lưới an toàn cuối.
- endpoints: 
- alternative: 

### Lua script nguyên tử trong Redis (redis-lua-atomic)

- how: Script Lua chạy nguyên tử trong Redis: nhiều lệnh không bị xen giữa.
- why: Rate limit và lease cần đọc-so sánh-ghi không race.
- applied here: ConnectionQuotaService.acquire chạy redis.eval(ACQUIRE_SCRIPT, 2, key...) để kiểm tra giới hạn và cấp lease nguyên tử, dùng redis TIME.
- endpoints: 
- alternative: 

### Socket.IO room theo user (socketio-rooms)

- how: Mỗi socket join room theo user id đã xác thực; tin được phát tới room.
- why: Gửi đúng người nhận và nhiều thiết bị cùng nhận.
- applied here: @WebSocketGateway({ namespace: '/signaling' }) đưa mỗi socket vào room user:{userId} của chính nó, không nhận room từ client.
- endpoints: 
- alternative: 

### Socket.IO Redis adapter (nhiều instance) (socketio-redis-adapter)

- how: Adapter Redis đồng bộ việc phát tin giữa nhiều instance gateway.
- why: Scale ngang gateway theo số kết nối.
- applied here: RedisAdapterService dựng createAdapter(pubClient, subClient) cho Socket.IO để nhiều replica gateway cùng phát tới socket của một user.
- endpoints: 
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: startLeaseRefresh tạo setInterval (unref) theo từng socket gọi connectionQuota.refresh(userId, leaseId); dừng ở handleDisconnect hoặc onModuleDestroy.
- endpoints: e50
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: Mỗi kết nối có setInterval gọi connectionQuota.refresh(userId, leaseId) để gia hạn lease cho tới khi disconnect.
- endpoints: 
- alternative: 

### Quota kết nối bằng lease (quota-lease)

- how: Mỗi kết nối giữ một lease trong Redis; vượt hạn mức thì từ chối.
- why: Giới hạn số kết nối mỗi user trên toàn cụm.
- applied here: WS_MAX_CONNECTIONS_PER_USER được thực thi bằng ConnectionQuotaService lease theo key signaling:connection-quota:{userId} dùng chung giữa các replica.
- endpoints: e50
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: Redis lỗi hoặc đang shutdown thì từ chối kết nối với CONNECTION_QUOTA_UNAVAILABLE thay vì cho qua; hết quota trả CONNECTION_LIMIT.
- endpoints: e50
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: Khi acquire lỗi hoặc disconnect, release lease là best-effort (.catch rỗng hoặc chỉ warn); TTL WS_CONNECTION_LEASE_MS là lưới an toàn khi Redis không khả dụng (áp cho e50).
- endpoints: e52
- alternative: 

### CORS allowlist chính xác (cors-allowlist)

- how: Chỉ origin trong danh sách chính xác mới được gọi cross-site.
- why: Cookie khác site chỉ an toàn khi origin được liệt kê đúng.
- applied here: CorsIoAdapter đặt cors.origin = allowedOrigins (từ CORS_ORIGINS) hoặc false khi danh sách rỗng, áp cho cả polling và WebSocket handshake.
- endpoints: e50
- alternative: 

### CORS allowlist chính xác (cors-allowlist)

- how: Chỉ origin trong danh sách chính xác mới được gọi cross-site.
- why: Cookie khác site chỉ an toàn khi origin được liệt kê đúng.
- applied here: CorsIoAdapter nhận allowedOrigins từ env CORS_ORIGINS để chỉ cho phép origin trong allowlist ở handshake Socket.IO.
- endpoints: 
- alternative: 

### Liveness / readiness probe (health-readiness)

- how: Liveness và readiness probe cho orchestrator.
- why: K8s biết khi nào gửi traffic hoặc khởi động lại.
- applied here: /health/ready kiểm signaling.isReady() (subscriber ready, PSUBSCRIBE xong, quota Redis ready) và redisAdapter.isReady(), fail thì 503; /health và /health/live không kiểm Redis.
- endpoints: e1, e2, e3
- alternative: 

### Log có cấu trúc (pino) (structured-logging)

- how: Log có cấu trúc dạng JSON bằng pino thay vì chuỗi tự do.
- why: Truy vết và tìm kiếm log theo từng trường thay vì đọc chuỗi tự do.
- applied here: LoggerModule.forRootAsync dùng buildPinoHttpOptions với level theo LOG_LEVEL và pretty chỉ ở development để log pino có cấu trúc.
- endpoints: 
- alternative: 

