# Admin quản lý danh mục quà

id: flow:admin-gift-catalog
group: Vận hành (admin)
services: module:admin, module:gift

Admin xem, tạo và sửa quà trong catalog; giá kim cương là dữ liệu trong bảng gifts do Gift sở hữu, còn Admin điều phối quyền và ghi audit log cùng transaction. Client thường chỉ đọc catalog đang bật.

Admin gọi AdminService, AdminService gọi GiftService ghi bảng gifts và ghi audit log trong cùng một transaction; client chỉ đọc GET /gifts.

## Participants

- admin-console (client): Admin console
- web (client): Web client
- admin (service): Admin
- gift (service): Gift
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/gifts: xem toàn bộ catalog (cần quyền ManageGifts) (Controller gắn @RequireAdminPermission(AdminPermission.ManageGifts); mọi route REST đều qua JwtAuthGuard.) {apps/core-api/src/modules/admin/admin.controller.ts:356}
2. admin -> gift [call/di-call]: Gọi GiftService.listAllForAdmin {apps/core-api/src/modules/admin/admin.service.ts:271}
3. gift -> postgres [call/shared-db]: Đọc mọi quà kể cả đã tắt, sắp theo sortOrder, code {apps/core-api/src/modules/gift/gift.service.ts:96}
4. admin -> admin-console [return/http]: Trả danh sách quà đầy đủ (AdminGiftDto)
5. admin-console -> admin [call/http]: POST /admin/gifts: tạo quà mới (code, name, priceDiamond) {apps/core-api/src/modules/admin/admin.controller.ts:367}
6. admin -> postgres [call/shared-db]: Mở transaction cho tạo quà và audit {apps/core-api/src/modules/admin/admin.service.ts:275}
7. admin -> gift [call/di-call]: GiftService.createGift(manager, input) trên cùng manager {apps/core-api/src/modules/admin/admin.service.ts:276}
8. gift -> postgres [call/shared-db]: INSERT quà (active=true); trùng code bị uq_gifts_code chặn (Lỗi unique uq_gifts_code được dịch thành CODE_ALREADY_EXISTS (409), transaction rollback.) {apps/core-api/src/modules/gift/gift.service.ts:109}
9. admin -> postgres [call/shared-db]: Ghi admin_audit_logs gift.created cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:277}
10. admin -> admin-console [return/http]: Trả quà vừa tạo (AdminGiftDto)
11. admin-console -> admin [call/http]: PATCH /admin/gifts/:id: sửa giá, tên, thứ tự, bật/tắt {apps/core-api/src/modules/admin/admin.controller.ts:380}
12. admin -> postgres [call/shared-db]: Mở transaction cho sửa quà và audit {apps/core-api/src/modules/admin/admin.service.ts:296}
13. admin -> gift [call/di-call]: GiftService.updateGift(manager, giftId, input) {apps/core-api/src/modules/admin/admin.service.ts:297}
14. gift -> postgres [call/shared-db]: Đọc quà theo id (404 nếu không có) rồi lưu trường đổi (Không hard-delete: tắt quà bằng active=false để gift_events còn FK tới gifts.id.) {apps/core-api/src/modules/gift/gift.service.ts:140}
15. admin -> postgres [call/shared-db]: Ghi admin_audit_logs gift.updated cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:298}
16. admin -> admin-console [return/http]: Trả quà sau khi sửa (AdminGiftDto)
17. web -> gift [call/http]: GET /gifts: client chỉ đọc catalog đang bật {apps/core-api/src/modules/gift/gift.controller.ts:41}
18. gift -> postgres [call/shared-db]: Đọc quà active=true theo sortOrder, code {apps/core-api/src/modules/gift/gift.service.ts:89}
19. gift -> web [return/http]: Trả danh mục quà kèm giá do server quyết định

## Invariants

- Giá quà nằm ở bảng gifts do Gift sở hữu; client chỉ đọc qua GET /gifts nên không thể tự đặt giá, và Admin chỉ điều phối quyền rồi gọi GiftService.
- Admin truyền cùng manager cho GiftService và audit log, nên quà đổi và bản ghi audit cùng commit hoặc cùng rollback.
- Quà không bị xóa cứng vì gift_events tham chiếu FK tới gifts.id; muốn gỡ quà thì đặt active=false, GET /gifts công khai chỉ trả quà active.

## Failure modes

- when Hai admin tạo trùng code gần như cùng lúc: uq_gifts_code chặn, GiftService ném CODE_ALREADY_EXISTS (409), transaction rollback, không có audit log.
- when PATCH với id không tồn tại: GiftService ném GIFT_NOT_FOUND (404), transaction rollback, không ghi audit.

## Endpoints covered

- GET /gifts [module:gift] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/gift.md
- GET /admin/gifts [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/gifts [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- PATCH /admin/gifts/:id [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
