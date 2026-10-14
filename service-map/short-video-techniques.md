# Techniques used by short-video

id: techniques:short-video
module: module:short-video

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi route /videos/* được bảo vệ bởi APP_GUARD JwtAuthGuard khai báo toàn cục trong app.module.ts; user lấy từ @CurrentUser() AuthenticatedUser.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: Xoá comment chỉ cho phép tác giả comment hoặc tác giả video: so sánh comment.authorUserId và video.authorUserId với user.userId trong short-video.service.ts.
- endpoints: e11, e2
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ e1 dùng Idempotency-Key: videoUploadIdempotencyKey(user,key) lưu ở Video.idempotencyKey unique; replay cấp lại presigned URL của storageKey gốc, khác caption thì 409. Các route khác không nhận header.
- endpoints: e1
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: recordView/like/unlike/createComment/deleteComment bọc dataSource.transaction; deleteComment giảm commentCount mà không kiểm affected (khác Feed) nên hai request xoá song song có thể giảm hai lần.
- endpoints: e6, e7, e8, e9, e11
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: uq_video_reactions_video_user và uq_video_views_video_viewer chặn trùng HÀNG like/view; riêng viewCount đọc view.qualified rồi increment không có guard nên hai request song song cùng viewer có thể cộng hai lần.
- endpoints: e1, e6, e7
- alternative: 

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: Video chuyển uploading, processing, pending_review|published, removed|rejected|failed bằng transition() là UPDATE ... WHERE status = from; thua race trả 0 rows (finalize, autoHide, sweeper).
- endpoints: e2, e5, e13
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: rankScore là cột derive do VideoRankingService tính lại theo lô 1000 từ view/like/comment và time-decay; sort=ranked loại rankScore IS NULL nên video mới chưa có điểm chưa hiện ở tab ranked.
- endpoints: e12, e3
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: VideoRankingService và VideoSweeperService mỗi cái giữ một ManagedInterval (this.job) chạy theo intervalMs lấy từ config.
- endpoints: e12, e13
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: Cả job ranking và sweeper chạy với clusterSingleton: { dataSource } nên chỉ một instance trong cluster chạy mỗi tick, dùng khoá qua DataSource Postgres.
- endpoints: e12, e13
- alternative: 

### Sweeper dọn dữ liệu hết hạn (sweeper)

- how: Job quét dọn bản ghi hết hạn.
- why: Giải phóng tài nguyên và tránh ghi rác.
- applied here: VideoSweeperService (đăng ký trong ShortVideoModule) quét video kẹt uploading quá timeout để đánh failed và dọn object mồ côi.
- endpoints: e13
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: ThrottlerGuard đăng ký APP_GUARD toàn cục áp dụng cho các endpoint /videos/*; giới hạn mặc định theo cấu hình throttler của app.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: VideoSweeperService gọi storagePort.delete trong try/catch: lỗi chỉ log warn và thử lại ở tick sau, không làm hỏng việc đánh failed video kẹt uploading.
- endpoints: e13
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: Production dùng UnavailableVideoStorageAdapter/UnavailableVideoTranscodeAdapter; assertVideoProviderConfiguration còn ném lỗi lúc khởi động nếu NODE_ENV=production mà VIDEO_UPLOAD_ENABLED=true.
- endpoints: 
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: ShortVideoController gắn @ApiOperation, @ApiIdempotencyKeyHeader và DTO có @ApiPropertyOptional để sinh hợp đồng OpenAPI cho client.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: 

### Upload presigned thẳng lên object storage (presigned-upload)

- how: Server cấp URL ký sẵn; client upload thẳng lên object storage.
- why: Body lớn không đi qua core-api.
- applied here: Chỉ e1 phát presigned URL (VideoStoragePort.issueUploadUrl) và e13 gọi delete dọn object; adapter hiện là Dev (URL giả), chưa có vendor thật; body video không đi qua core-api.
- endpoints: e1
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: GET /videos dùng ListVideosQueryDto extends CursorPageQueryDto (sort recent/ranked) và listComments nhận CursorPageQueryDto, phân trang theo cursor seq.
- endpoints: e10, e3
- alternative: 

### Xoá mềm (soft-delete)

- how: Đánh dấu xoá thay vì xoá hàng.
- why: Giữ lịch sử và khôi phục được.
- applied here: VideoComment có cột deletedAt timestamptz; truy vấn danh sách lọc c.deletedAt IS NULL nên comment bị xoá chỉ ẩn, không xoá hàng.
- endpoints: e10, e11
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: getVideoOrThrow trả cùng VIDEO_NOT_FOUND 404 cho video không tồn tại, removed, rejected hoặc chưa published (với người không phải tác giả).
- endpoints: e4, e5, e6, e7, e8, e9, e10, e11
- alternative: 

