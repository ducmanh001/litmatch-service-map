# Techniques used by avatar

id: techniques:avatar
module: module:avatar

### Khởi tạo cấu hình lazy (other:lazy-init-config)

- how: getMyAvatar và equip tạo user_avatar_configs mặc định ở lần đầu qua initDefaultConfig; PK userId chặn race, thua unique thì đọc lại dòng bên thắng.
- why: getMyAvatar và equip tạo user_avatar_configs mặc định ở lần đầu qua initDefaultConfig; PK userId chặn race, thua unique thì đọc lại dòng bên thắng.
- applied here: getMyAvatar và equip tạo user_avatar_configs mặc định ở lần đầu qua initDefaultConfig; PK userId chặn race, thua unique thì đọc lại dòng bên thắng.
- endpoints: e5, e6
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1, e2, e3, e4, e5, e6, e7
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Giá lấy từ asset.priceDiamond trong DB lúc claim/buy; claim chỉ cho giá 0 và buy chỉ cho giá khác 0, client không gửi giá.
- endpoints: e3, e4
- alternative: 

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: equip chỉ cho trang bị item user đã sở hữu (itemRepo.exists theo userId và avatarAssetId) và đúng slot, trả ITEM_NOT_OWNED 403, chống IDOR trang bị item trả phí chưa mua.
- endpoints: e5
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ POST /avatar/items/:assetId/buy (e4) dùng @IdempotencyKey và avatarBuyIdempotencyKey = avatar:buy:{userId}:{key}; unique idempotency_key của Transaction ở Economy chặn trừ DIA hai lần.
- endpoints: e4
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: grantOwnership dùng insert orIgnore trên unique uq_user_avatar_items_pair nên claim/buy lặp không lỗi; AvatarAsset.code unique; PK userId của user_avatar_configs chặn race lazy-init.
- endpoints: e3, e4, e5, e6
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Chỉ initDefaultConfig (lazy-init ở e5, e6) chạy trong dataSource.transaction ghi sở hữu item free và config; bước equip cập nhật cột slot bằng configRepo.update ngoài transaction.
- endpoints: e5, e6
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1, e2, e3, e4, e5, e6, e7
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: `@ApiOperation` (vd catalog 'giá server là nguồn sự thật') và `@ApiOkResponse` mô tả hợp đồng OpenAPI của route avatar.
- endpoints: e1, e2, e3, e4, e5, e6, e7
- alternative: 

