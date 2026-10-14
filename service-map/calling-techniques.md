# Techniques used by calling

id: techniques:calling
module: module:calling

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Các endpoint join/end/like/get của calling đi qua JwtAuthGuard là APP_GUARD toàn cục.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Xác thực chữ ký/token của webhook (webhook-signature)

- how: Webhook phải được xác minh chữ ký/token của nhà cung cấp trên body gốc trước khi xử lý.
- why: Endpoint webhook công khai nên chữ ký là cách duy nhất chứng minh nguồn gửi.
- applied here: Webhook LiveKit gọi livekit.receiveWebhook(rawBody, authHeader) để xác thực chữ ký/token; lỗi thì ném DomainException CallingErrors.WEBHOOK_INVALID.
- endpoints: e7
- alternative: 

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Sau khi like, CallingService gọi friendService.areFriends(user.userId, partnerId) để trả cờ matched, cho biết hai bên đã thành bạn hay chưa.
- endpoints: e3, e4, e6
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: durationSeconds và mốc startedAt tính theo giờ server core-api (có trừ thời gian reconnect), không tin client hay LiveKit; ticker enforce 7 phút ở server.
- endpoints: e4, e7, e9
- alternative: 

### Capability token ngắn hạn (grant của LiveKit) (capability-token)

- how: Cấp token ngắn hạn mang sẵn quyền cụ thể (publish, subscribe) cho đúng phòng.
- why: Quyền được enforce ngay ở hệ thống media chứ không tin client.
- applied here: Join call gọi livekit.mintJoinToken(roomName, userId, CALLING_TOKEN_TTL_SECONDS) để cấp token LiveKit ngắn hạn làm quyền vào phòng.
- endpoints: e1, e2
- alternative: 

### Guard toàn cục + @Public() để mở (global-guard-public-optout)

- how: Guard JWT đăng ký toàn cục; route nào muốn mở phải gắn @Public() tường minh.
- why: An toàn theo mặc định: quên cấu hình thì bị chặn chứ không bị mở.
- applied here: LivekitWebhookController.handle gắn @Public() để mở khỏi JwtAuthGuard toàn cục vì LiveKit gọi bằng chữ ký riêng, không có bearer.
- endpoints: e7
- alternative: Gắn guard từng controller: dễ sót route.

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: CallingService chạy dataSource.transaction(async (manager) => ...) khi like/kết thúc call, đọc CallSession trong transaction.
- endpoints: e6, e1, e3, e4, e5, e9, e7
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Trong transaction like/end, manager.findOne(CallSession, { lock: { mode: 'pessimistic_write' } }) khoá dòng call để tránh đua giữa end, webhook và ticker.
- endpoints: e6, e1, e3, e4, e5, e9, e7
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Index uq_call_sessions_match_session unique trên matchSessionId; hai bên join đồng thời thì bên thua bắt isUniqueViolation và lấy call đã tạo.
- endpoints: e1, e2, e6
- alternative: 

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: CallSession đi pending->active (markJoined khi cả hai identity joined) ->ended (endById); trạng thái ended terminal, mọi nguồn end đều đi qua một transition có khoá.
- endpoints: e4, e7, e9
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: endById commit transaction rồi mới cleanupEndedCall (chốt MatchSession, deleteRoom, publish call.ended); chỉ lần justEnded mới dọn nên retry không bắn call.ended đôi.
- endpoints: e4, e5, e7, e8, e9
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: GET /calling/calls/:id là fallback của call.ended và còn reconcileCallWithLivekit để đưa pending sang active hoặc chốt ended khi webhook mất.
- endpoints: e3, e8
- alternative: 

### Timer ân hạn trước khi đóng/kết thúc (grace-timer)

- how: Hẹn giờ ân hạn trước khi đóng hoặc kết thúc (ví dụ host rớt mạng).
- why: Tránh đóng phòng vì mất kết nối thoáng qua.
- applied here: Rớt kết nối mở reconnectStartedAt; sweepReconnectTimeouts end call sau CALLING_RECONNECT_WINDOW_SECONDS, còn quay lại đủ hai bên thì dời startedAt bù thời gian gián đoạn.
- endpoints: e7, e9
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: CallTickerService khởi động ManagedInterval với clusterSingleton dataSource nên pg_try_advisory_xact_lock chỉ cho một replica chạy tick tại một thời điểm.
- endpoints: e9
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: CallTickerService dùng ManagedInterval (clusterSingleton) chạy sweepPending, sweepReconnectTimeouts và processActiveCalls (free_limit) mỗi CALLING_TICKER_INTERVAL_MS, phục vụ endpoint job e9.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Chỉ join match-session (e1) và join friend (e2) có @Throttle riêng 20 mỗi phút trên nền ThrottlerGuard toàn cục; các endpoint end/like/get/webhook dùng throttle mặc định.
- endpoints: e1, e2, e3, e4, e5, e6, e7
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: deleteRoom bắt lỗi chỉ log debug và publishRealtimeEvent nuốt lỗi (client có poll GET); NGOẠI LỆ: endVoiceSessionForCall chạy trước và không bắt lỗi, nếu ném thì deleteRoom và call.ended bị bỏ qua.
- endpoints: e8
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: CallingController khai báo @ApiOperation/summary cho join, end, like, get call để sinh hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getCall trả CALL_NOT_FOUND 404 gộp cho cả call không tồn tại lẫn người không phải member nên không dò được callId của người khác.
- endpoints: e3
- alternative: 

