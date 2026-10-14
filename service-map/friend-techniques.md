# Techniques used by friend

id: techniques:friend
module: module:friend

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Follow, nhắn tin và tạo conversation từ profile kiểm tra quan hệ: `areFriends` dùng `canonicalPair` và `assertNotBlocked(...)` chặn khi bị block.
- endpoints: e11, e2, e3, e1, e4
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Ngày streak và first-contact tính theo ngày UTC của server (todayUtc, now() AT TIME ZONE 'UTC'), không nhận giờ client nên không spoof được múi giờ.
- endpoints: e8, e11, e15
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ e11 dùng Idempotency-Key: Message.idempotencyKey = `friend:msg:{userId}:{key}`, unique ở migration uq_messages_idempotency_key. Replay trả message cũ nhưng vẫn chạy lại publish realtime + notification; streak không tăng đôi nhờ guard ngày.
- endpoints: e11
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: `ensureFriendship(manager: EntityManager, ...)` và `lockProfileForChatInManager(manager, ...)` nhận manager để chạy chung transaction với bên gọi.
- endpoints: e4, e1
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: uq_friendships_pair và uq_conversations_pair (userLowId < userHighId, CHECK canonical ở migration) cùng orIgnore() (ON CONFLICT DO NOTHING) trong ensureFriendship/ensureConversationInManager nên tạo đôi hoặc mở chat song song không lỗi, không trùng.
- endpoints: e11, e4
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Tạo conversation từ profile và `recordActivity` của streak chạy trong `transaction(async (manager) => ...)` để ghi nguyên tử.
- endpoints: e4, e11
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: `lock: { mode: 'pessimistic_write' }` khoá dòng profile khi tạo conversation, và `setLock('pessimistic_write')` khoá `ConversationStreak` khi tính streak.
- endpoints: e4, e11
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: getDisplayStreak suy streak khi đọc (isActive khi gapDays <= 2, quá hạn hiện current=0 dù DB chưa reset); GET /friends tính lại unreadCount, canCall, muted từ DB mỗi lần.
- endpoints: e8, e5
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: Entity `Message` ghi rõ APPEND-ONLY: không có `updatedAt`, không extend `BaseAppEntity`.
- endpoints: 
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: sendMessage lưu message (transaction riêng) rồi mới publishRealtimeEvent friend.message tới sender và partner; lỗi publish bị nuốt, REST polling là fallback, không dùng outbox.
- endpoints: e11, e12
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: GET /conversations/:id/messages được web poll mỗi FRIEND_MESSAGES_REFETCH_INTERVAL_MS = 5_000 làm đường dự phòng khi socket rớt hoặc publish Redis lỗi.
- endpoints: e7
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: `StreakWarningJob` giữ `new ManagedInterval()` và `job.start(...)` với `intervalMs` từ config để lặp kiểm tra cảnh báo streak.
- endpoints: e15
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: `StreakWarningJob` start `ManagedInterval` với `clusterSingleton: { dataSource: this.dataSource }` nên chỉ một replica chạy nhờ pg advisory lock.
- endpoints: e15
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: Notification và push sau khi gửi tin chạy transaction riêng và best-effort, lỗi không làm fail `sendMessage`; `StreakWarningJob` cũng best-effort.
- endpoints: e11, e12, e13, e14
- alternative: 

### Hạn mức theo ngày (daily-quota)

- how: Hạn mức theo ngày lưu ở DB.
- why: Giới hạn lượt dùng miễn phí mỗi ngày để chặn lạm dụng mà vẫn cho thử tính năng; hạn mức lưu ở DB nên bền qua restart.
- applied here: Ngưỡng PROFILE_DIRECT_MESSAGE_DAILY_FIRST_CHAT_THRESHOLD: số ProfileChatContact của profile trong ngày UTC; chạm ngưỡng thì getActions đặt requiresGift và openConversation trả 402.
- endpoints: e1, e4
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Controller gắn `@ApiOperation`/`@ApiOkResponse` cho từng route (vd `GET friends`, `:profileUserId/actions`) làm hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: `GET /conversations/:id/messages` nhận `@Query() query: CursorPageQueryDto` và trả `MessagesPageDto` phân trang cursor.
- endpoints: e7
- alternative: 

### Cột JSONB (jsonb-columns)

- how: Cột JSONB lưu dữ liệu linh hoạt.
- why: Cấu trúc thay đổi thường xuyên.
- applied here: Cột `attachment` của `Message` kiểu `jsonb` nullable lưu `MessageAttachment` (ảnh, story_reply).
- endpoints: 
- alternative: 

### Schema chỉ đổi bằng migration (synchronize tắt) (migration-only-schema)

- how: Schema chỉ đổi bằng migration; synchronize tắt.
- why: Thay đổi schema có kiểm soát và rollback được.
- applied here: Index idx_messages_conversation_created (INCLUDE sender_user_id) khai synchronize:false vì chỉ migration 1756200000000 tạo; uq_messages_idempotency_key và chk_conversations_canonical cũng chỉ có ở migration friend-chat, entity không khai.
- endpoints: 
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getConversationForMember gộp 'không tồn tại' và 'không phải thành viên' thành FRIEND_CONVERSATION_NOT_FOUND 404; block 2 chiều ở sendMessage cũng trả cùng mã; GET /friends/:id/conversation trả NOT_FRIEND.
- endpoints: e6, e7, e8, e9, e10, e11
- alternative: 

