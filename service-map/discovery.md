# Service map: Discovery

id: service-map:discovery
module: module:discovery
deployable: core-api
confidence: high

Duyệt người dùng và Nearby ẩn vị trí

Sở hữu user_locations (toạ độ đã quantize ~500m) và discovery_settings (opt-in nearbyVisible), cộng logic browse/nearby: ghép bộ lọc, loại block/report, jitter khoảng cách, rate limit. Không sở hữu bảng users/privacy (module user), không sở hữu block/report (safety), không có logic mời Voice/Soul Match (matching).

Owns: bảng user_locations (latQuantized, lonQuantized, updatedAt); bảng discovery_settings (nearbyVisible mặc định false); Redis counter discovery:location-update:count:{userId} và discovery:nearby-query:count:{userId} (TTL 3600s)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/discovery/browse | jwt | Duyệt user theo gender/tuổi (cursor createdAt,id); loại self + block/report 2 chiều + hide_profile + guest/banned; trả DiscoveryCardDto kèm ageBucket, ageMin>ageMax trả 422. |
| e2 | rest | PUT | /api/v1/discovery/nearby/location | jwt | Ghi vị trí hiện tại, quantize ~500m ngay lúc ghi (không lưu toạ độ thô); rate limit DISCOVERY_LOCATION_UPDATE_RATE_LIMIT_PER_HOUR, vượt trả 429. |
| e3 | rest | PUT | /api/v1/discovery/nearby/visible | jwt | Bật/tắt opt-in Nearby; tắt thì xoá user_locations cùng transaction với cập nhật discovery_settings. |
| e4 | rest | GET | /api/v1/discovery/nearby | jwt | Liệt kê người gần: yêu cầu caller đã opt-in và vị trí còn tươi (403/409 nếu không), rate limit truy vấn, chỉ trả distanceBucket đã jitter, không trả toạ độ/km. |

## Coordination edges (why this service talks to others)

### lấy danh sách ẩn nhau (c1) with safety [service, out, di-call]

- endpoints: e1, e4
- what: SafetyService.getHiddenUserIds(userId): hợp các user bị block (2 chiều, đang active) với mọi user từng có report với caller (cả hai hướng, từ bảng reports); kết quả được gộp vào excludeUserIds/candidate loại trừ.
- why: Block/report thuộc Safety (nguồn sự thật), Discovery không được tự dựng lại luật; người đã block hoặc đã report nhau phải biến mất vĩnh viễn khỏi màn duyệt lặp lại nhiều lần/ngày (khác cooldown rematch), nên mỗi request phải hỏi Safety.
- if missing: Người dùng bị block/report vẫn hiện trong browse và Nearby của nhau, mở đường quấy rối và lộ vị trí gần đúng cho người đã chặn.
- evidence: apps/core-api/src/modules/discovery/discovery.service.ts:49, apps/core-api/src/modules/discovery/nearby.service.ts:174
- review: sound

### lọc user hợp lệ + riêng tư (c2) with user [service, out, di-call]

- endpoints: e1, e4
- what: UserService.browsePage(filter, limit, after) cho browse; UserService.findActiveByIds(ids, {gender, ageMin, ageMax, excludeGuests}) và PrivacySettingsService.findForUsers(ids) cho Nearby (lấy showDistance).
- why: Bảng users và luật 'user hợp lệ để hiện' (active, không guest nếu DISCOVERY_GUEST_VISIBLE=false, tuổi/gender, hide_profile) thuộc module user; gọi qua public API giữ một định nghĩa duy nhất và đảm bảo tôn trọng cài đặt riêng tư của đối phương (showDistance).
- if missing: Discovery phải tự query users và tự chế luật hiển thị, dễ lộ user banned/guest hoặc bỏ qua showDistance/hide_profile.
- evidence: apps/core-api/src/modules/discovery/discovery.service.ts:63, apps/core-api/src/modules/discovery/nearby.service.ts:188, apps/core-api/src/modules/discovery/nearby.service.ts:200
- review: sound

### rate limit vị trí/truy vấn (c3) with redis [infra, out, redis-state]

- endpoints: e2, e4
- what: checkRateLimit (Redis INCR+EXPIRE atomic) trên key discovery:location-update:count:{userId} và discovery:nearby-query:count:{userId}, cửa sổ 3600s, ngưỡng lấy từ env DISCOVERY_*_RATE_LIMIT_PER_HOUR.
- why: Chống trilateration: kẻ tấn công đổi vị trí và truy vấn nhiều lần để định vị chính xác một người từ distanceBucket; đếm phải atomic và dùng chung giữa các instance core-api nên đặt ở Redis, không thể giữ trong bộ nhớ process.
- if missing: Có thể quét liên tục nhiều vị trí giả để triangulate nơi ở của người khác, hoặc spam ghi vị trí; mỗi instance đếm riêng nên giới hạn thực tế bị nhân lên.
- evidence: apps/core-api/src/modules/discovery/discovery.module.ts:34, apps/core-api/src/modules/discovery/nearby.service.ts:76, apps/core-api/src/modules/discovery/nearby.service.ts:128
- review: sound

### ghi vị trí + opt-in (c4) with postgres [infra, out, shared-db]

- endpoints: e2, e3, e4
- what: Upsert user_locations (lat/lon quantized, updated_at), upsert discovery_settings.nearby_visible, DELETE user_locations khi tắt visible (transaction); đọc candidate bằng bounding-box SQL JOIN discovery_settings và user_privacy_settings.
- why: Trạng thái opt-in và vị trí phải bền vững và nhất quán: tắt visible phải xoá toạ độ cùng transaction để không còn dữ liệu vị trí của người đã rút lui; độ tươi dựa vào updated_at để vị trí cũ tự biến mất khi đọc.
- if missing: Không có opt-in thật sự (reciprocity), toạ độ cũ còn tồn tại sau khi user tắt Nearby, hoặc kết quả nearby không lọc được theo độ tươi.
- evidence: apps/core-api/src/modules/discovery/entities/discovery-setting.entity.ts:13, apps/core-api/src/modules/discovery/entities/user-location.entity.ts:14
- review: sound

### JOIN user_privacy_settings (c50) with user [service, out, shared-db]

- endpoints: e4
- what: queryCandidates chạy SQL thô JOIN user_privacy_settings (LEFT JOIN, COALESCE(hide_profile,false)=false) để loại những user đã bật hideProfile khỏi candidate Nearby ngay ở DB.
- why: Prefilter bounding-box giới hạn candidate bằng LIMIT DISCOVERY_NEARBY_CANDIDATE_CAP; nếu loại hide_profile muộn ở tầng app thì các hàng bị ẩn vẫn chiếm chỗ trong cap và đẩy người hợp lệ ra ngoài. Đây là coupling trực tiếp với bảng của module user, bỏ qua public API.
- if missing: Hàng hide_profile sẽ bị loại ở bước sau vẫn chiếm chỗ trong LIMIT DISCOVERY_NEARBY_CANDIDATE_CAP nên người hợp lệ biến mất khỏi kết quả; việc lộ hideProfile vẫn được UserService.findActiveByIds chặn lần hai bằng NOT EXISTS user_privacy_settings.
- evidence: apps/core-api/src/modules/discovery/nearby.service.ts:318
- review: corrected — UserService.findActiveByIds cũng loại hide_profile bằng NOT EXISTS, nên hideProfile không lộ; SQL chỉ để cap không bị lấp.

### lấy GPS rồi opt-in (c51) with web [client, in, http]

- endpoints: e2, e3, e4
- what: apps/web lấy toạ độ bằng navigator.geolocation.getCurrentPosition rồi PUT nearby/location, sau đó PUT nearby/visible=true; chỉ khi đó mới gọi GET nearby (nếu vị trí hết hạn thì client phải ghi lại).
- why: Server không có nguồn vị trí nào khác ngoài toạ độ do trình duyệt gửi; thứ tự location rồi visible là bắt buộc vì listNearby trả 409 nếu chưa có vị trí tươi và 403 nếu chưa opt-in.
- if missing: Không có vị trí nên Nearby luôn trả 409/403; user không thể opt-in và không thấy ai.
- evidence: apps/web/src/features/discovery/components/nearby-opt-in.tsx:109
- review: sound

## Notes

- Code vs doc: doc § 2 nói Discovery 'không sở hữu bảng riêng' (W1) nhưng § 8 và code có 2 bảng user_locations/discovery_settings; chỉ là doc ghép 2 giai đoạn, code là sự thật.
- Nearby có quy tắc loại trừ không nằm trong doc § 8.2: query SQL thô trong NearbyService.queryCandidates JOIN trực tiếp bảng user_privacy_settings (của module user) để loại hide_profile=true; và PrivacySettingsService.findForUsers quyết showDistance (false thì distanceBucket = null). Đây là coupling shared-db với bảng của module user, không đi qua public API của user.
- setLocation KHÔNG kiểm tra nearbyVisible: user chưa opt-in vẫn ghi được vị trí (chỉ bị ẩn nhờ JOIN discovery_settings.nearby_visible = true khi người khác truy vấn). Tắt visible mới xoá hàng user_locations.
- Discovery không có inbound caller trong module khác (dossier); CTA mời Voice/Soul Match ở UI gọi API của matching, không đi qua Discovery.
- DISCOVERY_REDIS là client ioredis riêng của module (createCoreRedisClient với REDIS_URL), đóng ở onModuleDestroy qua closeCoreRedisClient.
