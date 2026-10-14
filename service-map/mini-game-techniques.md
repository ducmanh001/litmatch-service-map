# Techniques used by mini-game

id: techniques:mini-game
module: module:mini-game

### UPDATE có điều kiện làm chốt chặn (other:conditional-update-guard)

- how: submitMove dùng UPDATE ... WHERE id AND <move> IS NULL AND status=waiting_moves; affected=0 trả 409 MOVE_ALREADY_SUBMITTED nên move không đổi được dù double-submit/2 instance.
- why: submitMove dùng UPDATE ... WHERE id AND <move> IS NULL AND status=waiting_moves; affected=0 trả 409 MOVE_ALREADY_SUBMITTED nên move không đổi được dù double-submit/2 instance.
- applied here: submitMove dùng UPDATE ... WHERE id AND <move> IS NULL AND status=waiting_moves; affected=0 trả 409 MOVE_ALREADY_SUBMITTED nên move không đổi được dù double-submit/2 instance.
- endpoints: e2
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Tạo ván oẳn tù tì chỉ với bạn bè: `friendService.areFriends(userId, friendUserId)` sai thì ném `MiniGameErrors.NOT_FRIEND`.
- endpoints: e1
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Tạo ván chạy trong `this.dataSource.transaction(async (manager) => ...)` lưu `MiniGameSession` cùng một đơn vị công việc.
- endpoints: e1, e2, e3
- alternative: 

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: MiniGameSession đi waiting_moves -> resolved|cancelled; submitMove chỉ UPDATE khi status=waiting_moves, cancel trả 409 nếu đã resolved, no-op nếu đã cancelled.
- endpoints: e2, e3
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Race tạo ván: bắt `isUniqueViolation(err)` rồi đọc lại `findWaitingByPair` để phân biệt replay idempotent với 409 cặp khác.
- endpoints: e1
- alternative: 

### Redis pub/sub fanout (at-most-once, không replay) (redis-pubsub-fanout)

- how: core-api PUBLISH lên kênh realtime:user:{id}; gateway PSUBSCRIBE rồi đẩy xuống socket; không lưu, không replay.
- why: Sự kiện chỉ là gợi ý còn REST là nguồn sự thật nên at-most-once là đủ và rất rẻ.
- applied here: Event minigame.session.started/resolved được gửi cho cả hai user bằng `publishRealtimeEvent(this.redis, ...)` trong `Promise.all` qua Redis pub/sub.
- endpoints: e5, e6
- alternative: Redis Streams hoặc Kafka bền và replay được nhưng cần consumer group + ack: thừa với tin gợi ý.

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: minigame.session.started publish sau transaction tạo ván; minigame.session.resolved chỉ publish khi submitMove trả resolved != null, sau khi transaction đã xong.
- endpoints: e5, e6
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: GET /mini-game/sessions/:id trả cùng payload realtime để client poll khi minigame.* không tới.
- endpoints: e4
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1, e2, e3, e4
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Controller dùng `@ApiOperation` mô tả mỗi route (vd POST sessions idempotent) làm hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getSessionForParticipant gộp ván không tồn tại và không phải thành viên thành 404 NOT_FOUND.
- endpoints: e2, e3, e4
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: Khi mới một bên nộp move, submitMove không resolve và không publish gì để không lộ trạng thái nộp của đối phương; chỉ resolved mới gửi lowMove/highMove cho cả hai.
- endpoints: e2, e5
- alternative: 

