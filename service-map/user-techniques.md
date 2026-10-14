# Techniques used by user

id: techniques:user
module: module:user

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: JwtAuthGuard đăng ký APP_GUARD toàn cục nên mọi route /users/* yêu cầu access token JWT, user lấy từ @CurrentUser().
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: Các route /users/me và /users/me/privacy luôn dùng user.userId từ JWT (CurrentUser), không nhận id trong path hay body, nên chỉ đọc/ghi hồ sơ và privacy của chính mình.
- endpoints: e1, e2, e3, e4
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: User có cột isGuest và status (UserStatus), DTO profile trả isGuest; các thuộc tính này làm đầu vào cho chính sách theo thuộc tính.
- endpoints: 
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: PrivacySettingsService cập nhật bằng settingRepo.upsert theo userId, dựa vào khoá duy nhất để tạo hoặc cập nhật nguyên tử.
- endpoints: e4
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: isOnline không lưu trong Postgres: e5 tính từ showOnlineStatus (privacy) AND lease ZSET realtimePresenceKey do signaling-gateway ghi; short-circuit nên không chạm Redis khi user tắt hiển thị.
- endpoints: e5
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: UserService.createWithManager(manager, input) nhận EntityManager để Auth tạo user cùng transaction với AuthIdentity, tránh user mồ côi.
- endpoints: 
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Khoá pessimistic_write dùng ở lockRoleAssignmentContext (admin + target) khi đổi role; ban/unban (setStatus) chỉ findOneBy rồi save trong transaction của admin, không khoá hàng. Không phải endpoint e1-e6.
- endpoints: 
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: ThrottlerGuard được đăng ký APP_GUARD toàn cục trong app.module nên áp rate limit mặc định cho mọi route user.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: RedisPresenceReader.isOnline bắt mọi lỗi Redis và trả false; e5 không bao giờ báo online giả khi Redis sập.
- endpoints: e5
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: RedisPresenceReader.isOnline fail closed: Redis lỗi thì trả không online thay vì lộ trạng thái online giả.
- endpoints: 
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: UserController dùng @ApiOperation và @ApiOkResponse({ type: MyProfileDto }) cho hợp đồng OpenAPI của các route user.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: 

### Cột JSONB (jsonb-columns)

- how: Cột JSONB lưu dữ liệu linh hoạt.
- why: Cấu trúc thay đổi thường xuyên.
- applied here: Cột interests của User khai báo @Column({ type: 'jsonb', nullable: true }) lưu mảng sở thích.
- endpoints: 
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: GET /users/:id trả PublicProfileDto chỉ gồm id, nickname, gender, avatarId, interests; không có birthDate, region, seeking* hay isGuest (các trường đó chỉ nằm trong MyProfileDto).
- endpoints: e6
- alternative: 

