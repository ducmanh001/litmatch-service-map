# Techniques used by mood

id: techniques:mood
module: module:mood

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Các endpoint /mood/* đi qua JwtAuthGuard là APP_GUARD toàn cục.
- endpoints: e1, e2, e3, e4, e5
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ POST/DELETE /mood/status (e4,e5) bắt Idempotency-Key: moodSetIdempotencyKey/moodClearIdempotencyKey prefix riêng set/clear, unique ở DB, unique violation thì đọc lại dòng cũ.
- endpoints: e4, e5
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: MoodStatusEvent.idempotencyKey unique là chốt cuối cho retry set/clear (e4,e5); MoodPreset.code unique chỉ giữ catalog không trùng mã, không chặn request trùng.
- endpoints: e4, e5
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: Mood hiện tại không lưu cột: deriveCurrentMood lấy dòng event mới nhất theo createdAt DESC, trả null nếu kind=clear hoặc expiresAt <= now.
- endpoints: e2, e3
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: setMood/clearMood chỉ insert dòng mood_status_events mới (set/clear), không update hay xoá dòng cũ; re-set cùng preset vẫn tạo dòng mới để refresh TTL.
- endpoints: e4, e5
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: MoodStatusEvent là log append-only set/clear; mood hiện tại derive khi đọc từ dòng mới nhất (getCurrentMood) không phải clear và chưa quá expiresAt.
- endpoints: 
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: Hết hạn theo MOOD_STATUS_TTL_HOURS được xét lúc đọc bằng so expiresAt với giờ server, không cron dọn nên không cần worker.
- endpoints: e2, e3
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Các endpoint mood chịu ThrottlerGuard là APP_GUARD toàn cục trong app.module.ts, dùng hạn mức mặc định.
- endpoints: e1, e2, e3, e4, e5
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: MoodController khai báo @ApiOperation/@ApiOkResponse({ type: MoodPresetDto, isArray: true }) cho presets và status để sinh OpenAPI.
- endpoints: e1, e2, e3, e4, e5
- alternative: 

