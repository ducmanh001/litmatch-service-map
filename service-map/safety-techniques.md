# Techniques used by safety

id: techniques:safety
module: module:safety

### Ghi idempotent theo dòng mới nhất (other:idempotent-write)

- how: block khi dòng mới nhất đã là blocked và unblock khi chưa blocked đều return sớm không ghi dòng thừa nên gọi lặp (retry/double tap) vô hại, trả 204.
- why: block khi dòng mới nhất đã là blocked và unblock khi chưa blocked đều return sớm không ghi dòng thừa nên gọi lặp (retry/double tap) vô hại, trả 204.
- applied here: block khi dòng mới nhất đã là blocked và unblock khi chưa blocked đều return sớm không ghi dòng thừa nên gọi lặp (retry/double tap) vô hại, trả 204.
- endpoints: e2, e3
- alternative: 

### Cooldown trừ điểm theo cặp reporter-target (other:pair-cooldown)

- how: Cùng một reporter report lại cùng target trong SAFETY_REPORT_COOLDOWN_DAYS vẫn được ghi làm bằng chứng nhưng trustPenaltyApplied = 0, chống vote-kick lặp.
- why: Cùng một reporter report lại cùng target trong SAFETY_REPORT_COOLDOWN_DAYS vẫn được ghi làm bằng chứng nhưng trustPenaltyApplied = 0, chống vote-kick lặp.
- applied here: Cùng một reporter report lại cùng target trong SAFETY_REPORT_COOLDOWN_DAYS vẫn được ghi làm bằng chứng nhưng trustPenaltyApplied = 0, chống vote-kick lặp.
- endpoints: e1
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Các route /safety/* (report, block, unblock, trạng thái block) được JwtAuthGuard toàn cục bảo vệ, user lấy từ @CurrentUser().
- endpoints: e1, e2, e3, e4
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: GET /safety/blocks/:targetUserId gọi safetyService.isBlocked(user.userId, targetUserId), kiểm tra quan hệ block một chiều giữa hai user.
- endpoints: e4
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: Trạng thái đang block không lưu cột riêng mà suy ra từ dòng blocks mới nhất theo cặp (latestAction ORDER BY createdAt DESC); block/unblock chỉ append dòng mới.
- endpoints: e2, e3, e4
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Báo cáo user bọc trong this.dataSource.transaction(async (manager) => ...) để insert report và cập nhật trust score cùng một giao dịch.
- endpoints: e1
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Trong transaction report, findOne User với lock pessimistic_write trên hàng target để serialize trừ điểm, tránh vượt daily cap khi báo cáo đồng thời.
- endpoints: e1
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: Block là log append-only theo cặp (blocker, blocked), trạng thái hiện tại là dòng mới nhất; Report cũng append-only và ghi trustPenaltyApplied.
- endpoints: 
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: SafetyService.setReportStatus(manager: EntityManager, ...) nhận manager từ caller như UserService.banUser để cập nhật status chung transaction.
- endpoints: 
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Khi insert report video trùng, bắt lỗi bằng isUniqueViolation rồi findOneByOrFail báo cáo đã có, thay vì tạo bản trùng.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Chỉ POST /safety/reports (e1) có @Throttle limit 10 mỗi 60 phút/tài khoản, bổ sung cho cooldown theo cặp và daily cap; block/unblock/get dùng ThrottlerGuard toàn cục mặc định.
- endpoints: e1, e2, e3, e4
- alternative: 

### Hạn mức theo ngày (daily-quota)

- how: Hạn mức theo ngày lưu ở DB.
- why: Giới hạn lượt dùng miễn phí mỗi ngày để chặn lạm dụng mà vẫn cho thử tính năng; hạn mức lưu ở DB nên bền qua restart.
- applied here: Trần trừ trust score mỗi target theo SAFETY_TRUST_PENALTY_DAILY_CAP: cộng trustPenaltyApplied trong 24h, penalty = min(PER_REPORT, remainingCap); report vượt trần vẫn ghi với penalty 0.
- endpoints: e1
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: SafetyController dùng @ApiOperation và @ApiCreatedResponse({ type: ReportDto }) mô tả hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4
- alternative: 

### Schema chỉ đổi bằng migration (synchronize tắt) (migration-only-schema)

- how: Schema chỉ đổi bằng migration; synchronize tắt.
- why: Thay đổi schema có kiểm soát và rollback được.
- applied here: Index idx_blocks_blocked_blocker_created khai báo { synchronize: false } vì covering INCLUDE(action) chỉ tạo đúng bằng SQL migration.
- endpoints: 
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: GET /safety/blocks/:targetUserId chỉ gọi isBlocked(caller, target) một chiều nên không lộ việc target đã block ngược lại caller.
- endpoints: e4
- alternative: 

