# Techniques used by media

id: techniques:media
module: module:media

### Kiểm magic bytes ảnh (other:magic-byte-sniffing)

- how: Asset do e1 cấp chỉ thành ready khi resolveImageUrl đọc 4096 byte đầu và detectImageMime (png/jpeg/gif/webp) khớp contentType đã khai báo; sai thì xoá object và trả ASSET_NOT_READY 422.
- why: Asset do e1 cấp chỉ thành ready khi resolveImageUrl đọc 4096 byte đầu và detectImageMime (png/jpeg/gif/webp) khớp contentType đã khai báo; sai thì xoá object và trả ASSET_NOT_READY 422.
- applied here: Asset do e1 cấp chỉ thành ready khi resolveImageUrl đọc 4096 byte đầu và detectImageMime (png/jpeg/gif/webp) khớp contentType đã khai báo; sai thì xoá object và trả ASSET_NOT_READY 422.
- endpoints: e1
- alternative: 

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Mọi endpoint của service được bảo vệ bởi `JwtAuthGuard` đăng ký qua `APP_GUARD` toàn cục trong app.module.ts, nên cần bearer JWT hợp lệ mới vào được controller.
- endpoints: e1
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: `assertNotGuest(user)` ném `MediaErrors.GUEST_FORBIDDEN` khi `user.isGuest`, guest không được xin upload ảnh.
- endpoints: e1
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: assetId do e1 gắn với ownerUserId; resolveImageUrl tìm theo {id, ownerUserId} và kiểm purpose, sai thì ASSET_NOT_FOUND 404 (không phân biệt asset không có hay của người khác).
- endpoints: e1
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Cột `storageKey` của `ImageAsset` có `unique: true` đảm bảo mỗi object storage chỉ gắn một asset.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Giới hạn tốc độ qua `ThrottlerGuard` đăng ký `APP_GUARD` toàn cục, chạy trước JwtAuthGuard; áp cho mọi route của service.
- endpoints: e1
- alternative: 

### Upload presigned thẳng lên object storage (presigned-upload)

- how: Server cấp URL ký sẵn; client upload thẳng lên object storage.
- why: Body lớn không đi qua core-api.
- applied here: createUploadIntent trả presigned PUT (getSignedUrl, ký ContentType, TTL MEDIA_UPLOAD_URL_TTL_SECONDS); sizeBytes không được ký nên kích thước/MIME được kiểm lại bằng head + readPrefix khi resolveImageUrl.
- endpoints: e1
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: `@ApiOperation` và `@ApiCreatedResponse({ type: ImageUploadIntentDto })` mô tả hợp đồng OpenAPI của upload-intent.
- endpoints: e1
- alternative: 

### Quarantine → promote (kiểm trước khi công bố) (quarantine-promote)

- how: Object upload nằm ở khu cách ly, kiểm xong mới chuyển sang key công khai.
- why: Không công bố file chưa kiểm.
- applied here: resolveImageUrl: assertUploadedImage (head + magic bytes) rồi storage.promote (CopyObject sang images/{owner}/{asset} + xoá key cũ), status sang ready; với MEDIA_STORAGE_PROVIDER=dev bước kiểm/promote bị bỏ qua.
- endpoints: 
- alternative: 

