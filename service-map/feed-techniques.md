# Techniques used by feed

id: techniques:feed
module: module:feed

### UPDATE có điều kiện chống trừ đôi (other:conditional-update-guard)

- how: deleteComment update { id, deletedAt: IsNull() } và chỉ giảm commentCount khi affected = 1, nên hai request xoá song song không trừ counter hai lần.
- why: deleteComment update { id, deletedAt: IsNull() } và chỉ giảm commentCount khi affected = 1, nên hai request xoá song song không trừ counter hai lần.
- applied here: deleteComment update { id, deletedAt: IsNull() } và chỉ giảm commentCount khi affected = 1, nên hai request xoá song song không trừ counter hai lần.
- endpoints: e13
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: Xem/xoá post, story kiểm `post.authorUserId !== user.userId` (và `story.authorUserId`); khác chủ thì phải qua kiểm block mới được truy cập.
- endpoints: e9, e10, e11, e12, e14, e15, e16, e13, e3, e5, e4
- alternative: 

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Feed kiểm `safetyService.isBlocked` hai chiều, `privacySettings.isHidden` và `FriendModule` (`areFriends`/`listFriendIds`) cho audience=friends.
- endpoints: e8, e9, e10, e11, e12, e14, e15, e16, e3, e5
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: assertNotGuest ném GUEST_FORBIDDEN 403 trong FeedService/StoryService khi tạo post/story/comment/like; controller chỉ bỏ qua resolveImageUrl cho guest chứ không tự chặn.
- endpoints: e1, e6, e11, e14
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ e6 (Post.idempotencyKey, unique index một phần WHERE NOT NULL) và e1 (Story.idempotencyKey unique) dùng header; e5 chuyển key sang Friend. Like/comment khử trùng bằng unique hoặc UPDATE có điều kiện, không dùng header.
- endpoints: e1, e5, e6
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Index `uq_reactions_post_user` (postId, userId) giới hạn mỗi user một reaction trên post; `Story.idempotencyKey` có `unique: true`.
- endpoints: e6, e14, e1, e3
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Comment và reaction ghi trong `dataSource.transaction`: lưu `Comment`, tăng `commentCount` và tạo notification cùng transaction.
- endpoints: e11, e13, e14, e15
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: Trong transaction comment, `notificationService.createWithManager(manager, {...})` tạo notification chung manager với bài viết.
- endpoints: e11, e14
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: Story hết hạn bị lọc lúc đọc bằng s.expiresAt > :now (ring) và expiresAt <= now (getStoryOrThrow); StorySweeperService chỉ dọn rác, không quyết định correctness.
- endpoints: e2, e3, e5
- alternative: 

### Sweeper dọn dữ liệu hết hạn (sweeper)

- how: Job quét dọn bản ghi hết hạn.
- why: Giải phóng tài nguyên và tránh ghi rác.
- applied here: `StorySweeperService` (provider trong `FeedModule`) hard-delete story hết hạn theo lô qua `ManagedInterval`.
- endpoints: e17
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: `StorySweeperService` giữ `new ManagedInterval()` lặp theo `STORY_SWEEPER_INTERVAL_MS`.
- endpoints: e17
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: `StorySweeperService` start `ManagedInterval` với `clusterSingleton: { dataSource: this.dataSource }` nên chỉ một instance toàn cluster dọn story.
- endpoints: e17
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: `@ApiOperation` và `@ApiIdempotencyKeyHeader()` trên route đăng bài/story mô tả hợp đồng OpenAPI (guest bị chặn, TTL story).
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16
- alternative: 

### Xoá mềm (soft-delete)

- how: Đánh dấu xoá thay vì xoá hàng.
- why: Giữ lịch sử và khôi phục được.
- applied here: `Post` và `Comment` có cột `deletedAt` timestamptz nullable; xoá chỉ đánh dấu, danh sách lọc bỏ bản ghi đã xoá.
- endpoints: e7, e8, e9, e10, e11, e12, e14, e15, e16, e13
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: Feed, bài theo user và comment trả `PostsPageDto` có `nextCursor`, nhận `@Query() query: CursorPageQueryDto`.
- endpoints: e7, e8, e12
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getPostOrThrow và getStoryOrThrow trả cùng POST_NOT_FOUND/STORY_NOT_FOUND 404 cho không tồn tại, đã xoá/hết hạn, bị block, tác giả ẩn hồ sơ hoặc sai audience.
- endpoints: e3, e5, e9, e10, e11, e12, e14, e15, e16
- alternative: 

