# Techniques used by discovery

id: techniques:discovery
module: module:discovery

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Browse và nearby lọc user qua `safetyService.getHiddenUserIds(user.userId)` để loại những người đã block/report hai chiều.
- endpoints: e1, e4
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: listNearby chỉ cho caller đã opt-in: nếu discovery_settings.nearbyVisible=false trả 403 NEARBY_NOT_OPTED_IN; vị trí thiếu/hết hạn trả 409 NEARBY_LOCATION_MISSING.
- endpoints: e4
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: `setLocation` upsert `user_locations` bằng `ON CONFLICT (user_id) DO UPDATE`, dựa vào khoá unique theo user.
- endpoints: e2, e3
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: `setVisible` chạy `dataSource.transaction` upsert `discovery_settings` và xoá `user_locations` khi tắt, cùng một transaction.
- endpoints: e3
- alternative: 

### Lua script nguyên tử trong Redis (redis-lua-atomic)

- how: Script Lua chạy nguyên tử trong Redis: nhiều lệnh không bị xen giữa.
- why: Rate limit và lease cần đọc-so sánh-ghi không race.
- applied here: RedisRateLimitAdapter.consume chạy Lua RATE_LIMIT_CONSUME_LUA: INCR+EXPIRE nguyên tử, vượt hạn thì DECR lại nên lượt bị từ chối không tính.
- endpoints: e2, e4
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: Vị trí quá DISCOVERY_LOCATION_FRESHNESS_HOURS tự biến mất khỏi kết quả nhờ lọc ul.updated_at lúc đọc; không có cron dọn user_locations.
- endpoints: e4
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1, e2, e3, e4
- alternative: 

### Rate limit bằng Redis (Lua) (rate-limit-redis)

- how: Bộ đếm dùng chung trong Redis (Lua) giới hạn theo khoá nghiệp vụ.
- why: Giới hạn đúng khi có nhiều instance.
- applied here: checkRateLimit trên discovery:location-update:count:{userId} và discovery:nearby-query:count:{userId}, cửa sổ 3600s, ngưỡng DISCOVERY_*_RATE_LIMIT_PER_HOUR; vượt trả 429 NEARBY_RATE_LIMITED.
- endpoints: e2, e4
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: `@ApiOperation` trên `GET browse` mô tả bộ lọc gender/tuổi, loại block/report, làm hợp đồng OpenAPI.
- endpoints: e1, e2, e3, e4
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: `BrowseQueryDto` và `NearbyQueryDto` đều `extends CursorPageQueryDto` nên browse/nearby phân trang theo cursor.
- endpoints: e4
- alternative: 

### Truy vấn theo vị trí (nearby) (geo-query)

- how: Truy vấn khoảng cách địa lý để tìm người ở gần.
- why: Khám phá người ở gần cần tính khoảng cách địa lý ngay trong truy vấn thay vì lọc ở tầng ứng dụng.
- applied here: Nearby tính khoảng cách bằng `haversineDistanceKm` với toạ độ lượng tử hoá (`quantizeCoordinate`, `nearbyJitterKm`) và bán kính `DISCOVERY_NEARBY_RADIUS_KM`, không dùng PostGIS.
- endpoints: e4
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: Browse dùng cursor (createdAt,id) qua decodeBrowseCursor và UserService.browsePage(filter, limit, after), trả meta.nextCursor.
- endpoints: e1
- alternative: 

### Tối thiểu hoá dữ liệu trả về (ẩn danh, DTO công khai) (data-minimization)

- how: Chỉ trả trường cần thiết (DTO công khai, ẩn danh theo từng người nhận).
- why: Giảm rò rỉ dữ liệu: chỉ trả đúng các trường cần thiết cho người nhận thay vì cả bản ghi nội bộ.
- applied here: Toạ độ quantize ngay lúc ghi (DISCOVERY_LOCATION_QUANTIZE_DEGREES), Nearby chỉ trả distanceBucket đã jitter theo cặp-ngày và null nếu đối phương tắt showDistance; không trả toạ độ/km.
- endpoints: e2, e4
- alternative: 

