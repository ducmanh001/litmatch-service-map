# Khám phá người dùng và gần đây

id: flow:discovery-browse-and-nearby
group: Ghép đôi & gọi
services: module:discovery, module:safety, module:user

Người dùng duyệt danh sách theo giới tính/tuổi và tìm người gần bằng vị trí đã quantize. Discovery sở hữu vị trí và opt-in nearby nhưng phải hỏi Safety để loại người bị block/report và hỏi User để lấy hồ sơ active và quyền riêng tư.

Discovery ghép dữ liệu từ Safety (danh sách ẩn), User (hồ sơ, privacy), Redis (rate limit) và PostgreSQL (vị trí, opt-in) cho browse và nearby.

## Participants

- web (client): Web client
- discovery (service): Discovery
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL
- redis (infra): Redis

## Steps

1. web -> discovery [call/http]: GET /discovery/browse với bộ lọc gender/tuổi và cursor (Mọi route REST đều qua JwtAuthGuard toàn cục.)
2. discovery -> safety [call/di-call]: Lấy danh sách user bị ẩn (block/report) của người xem {apps/core-api/src/modules/discovery/discovery.service.ts:49}
3. discovery -> user [call/di-call]: Truy vấn trang hồ sơ, loại bản thân và user bị ẩn (Filter gồm excludeUserIds và excludeGuests theo DISCOVERY_GUEST_VISIBLE; Discovery sau đó gắn ageBucket.) {apps/core-api/src/modules/discovery/discovery.service.ts:63}
4. discovery -> web [return/http]: Trả trang thẻ người dùng kèm ageBucket và nextCursor
5. web -> discovery [call/http]: PUT /discovery/nearby/visible bật hoặc tắt hiển thị
6. discovery -> postgres [call/shared-db]: Upsert discovery_settings; nếu tắt thì xoá user_locations (Cả hai lệnh nằm trong cùng callback dataSource.transaction.) {apps/core-api/src/modules/discovery/nearby.service.ts:109}
7. web -> discovery [call/http]: PUT /discovery/nearby/location gửi toạ độ hiện tại
8. discovery -> redis [call/redis-state]: Kiểm tra giới hạn số lần cập nhật vị trí mỗi giờ {apps/core-api/src/modules/discovery/nearby.service.ts:76}
9. discovery -> postgres [call/shared-db]: Upsert user_locations bằng toạ độ đã quantize {apps/core-api/src/modules/discovery/nearby.service.ts:98}
10. web -> discovery [call/http]: GET /discovery/nearby lấy người ở gần
11. discovery -> redis [call/redis-state]: Kiểm tra giới hạn số lần truy vấn nearby mỗi giờ {apps/core-api/src/modules/discovery/nearby.service.ts:128}
12. discovery -> postgres [call/shared-db]: Đọc opt-in nearbyVisible và vị trí của chính người xem (Chưa opt-in thì lỗi 403; vị trí thiếu hoặc quá hạn thì lỗi 409.) {apps/core-api/src/modules/discovery/nearby.service.ts:144}
13. discovery -> safety [call/di-call]: Lấy danh sách user bị ẩn để loại khỏi ứng viên {apps/core-api/src/modules/discovery/nearby.service.ts:174}
14. discovery -> postgres [call/shared-db]: Truy vấn ứng viên: opt-in, không hide_profile, vị trí còn mới, trong khung bán kính (SQL join discovery_settings (nearby_visible=true) và user_privacy_settings (hide_profile=false), lọc updated_at theo DISCOVERY_LOCATION_FRESHNESS_HOURS, bounding box lat/lon, loại bản thân và user bị ẩn, LIMIT theo DISCOVERY_NEARBY_CANDIDATE_CAP; sau đó lọc haversine <= bán kính trong bộ nhớ.) {apps/core-api/src/modules/discovery/nearby.service.ts:314}
15. discovery -> user [call/di-call]: Lọc ứng viên còn active theo gender/tuổi/guest {apps/core-api/src/modules/discovery/nearby.service.ts:188}
16. discovery -> user [call/di-call]: Lấy quyền riêng tư (showDistance) của các user còn lại (Nếu showDistance=false thì distanceBucket trả null; khoảng cách được cộng jitter tất định theo cặp và ngày rồi mới chia bucket.) {apps/core-api/src/modules/discovery/nearby.service.ts:200}
17. discovery -> web [return/http]: Trả trang nearby chỉ có distance bucket, không có toạ độ

## Invariants

- Discovery phải hỏi Safety getHiddenUserIds trước khi truy vấn ứng viên để người bị block/report hai chiều không bao giờ xuất hiện trong browse hay nearby.
- Tắt nearby phải cập nhật discovery_settings và xoá user_locations trong cùng một transaction, nếu không có thể còn vị trí lưu khi người dùng đã rút opt-in.
- Hồ sơ, trạng thái active và quyền riêng tư thuộc module User nên Discovery chỉ lấy qua UserService/PrivacySettingsService thay vì đọc bảng users.

## Failure modes

- when Vượt giới hạn cập nhật vị trí hoặc truy vấn nearby mỗi giờ: Discovery ném NEARBY_RATE_LIMITED với HTTP 429 và không ghi hay truy vấn thêm.
- when Người xem chưa bật nearbyVisible: listNearby ném NEARBY_NOT_OPTED_IN với HTTP 403 trước khi gọi Safety hay User.
- when Người xem chưa có vị trí hoặc vị trí quá hạn: listNearby ném NEARBY_LOCATION_MISSING với HTTP 409.
- when ageMin lớn hơn ageMax hoặc cursor browse sai định dạng: browse ném FILTER_INVALID (422) hoặc CURSOR_INVALID (400) trước khi gọi Safety.
- when Toạ độ ngoài khoảng hợp lệ khi PUT /discovery/nearby/location: setLocation ném LOCATION_INVALID (422) trước khi kiểm tra rate limit Redis, không ghi gì.

## Endpoints covered

- GET /discovery/browse [module:discovery] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/discovery.md
- PUT /discovery/nearby/location [module:discovery] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/discovery.md
- PUT /discovery/nearby/visible [module:discovery] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/discovery.md
- GET /discovery/nearby [module:discovery] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/discovery.md
