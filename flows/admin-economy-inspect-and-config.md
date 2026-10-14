# Admin xem ví, giao dịch và chỉnh cấu hình kinh tế

id: flow:admin-economy-inspect-and-config
group: Vận hành (admin)
services: module:admin, module:economy

Admin xem ví, lịch sử giao dịch và catalog kinh tế rồi bật/tắt IAP product hoặc VIP plan; Admin chỉ điều phối và ghi audit, còn dữ liệu do Economy sở hữu nên không sửa ledger trực tiếp.

Admin gọi Economy qua DI để đọc ví, sổ cái, catalog; hai thao tác PATCH ghi cấu hình và audit log trong cùng một transaction.

## Participants

- admin-console (client): Admin console
- admin (service): Admin
- economy (service): Economy
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/economy/wallet/:userId xem ví user
2. admin -> economy [call/di-call]: getWallet(userId) chỉ đọc ví và VIP {apps/core-api/src/modules/admin/admin.service.ts:313}
3. admin -> admin-console [return/http]: Trả ví: balance và VIP
4. admin-console -> admin [call/http]: GET /admin/economy/users/:userId/transactions
5. admin -> economy [call/di-call]: listTransactions(userId, limit, cursor) đọc sổ cái {apps/core-api/src/modules/admin/admin.service.ts:326}
6. admin -> admin-console [return/http]: Trả trang giao dịch theo cursor
7. admin-console -> admin [call/http]: GET /admin/config/economy-catalog
8. admin -> economy [call/di-call]: listCatalogForAdmin() lấy cả IAP/VIP inactive {apps/core-api/src/modules/admin/admin.service.ts:367}
9. admin -> admin-console [return/http]: Trả catalog IAP product và VIP plan
10. admin-console -> admin [call/http]: PATCH /admin/config/iap-products/:productId bật/tắt
11. admin -> postgres [call/shared-db]: Mở transaction cho thao tác đổi cấu hình IAP {apps/core-api/src/modules/admin/admin.service.ts:375}
12. admin -> economy [call/di-call]: setIapProductActive(manager, productId, active) {apps/core-api/src/modules/admin/admin.service.ts:376}
13. admin -> postgres [call/shared-db]: Ghi audit economy.iap_product.updated cùng manager {apps/core-api/src/modules/admin/admin.service.ts:381}
14. admin -> admin-console [return/http]: Trả IAP product đã cập nhật
15. admin-console -> admin [call/http]: PATCH /admin/config/vip-plans/:id bật/tắt gói VIP
16. admin -> postgres [call/shared-db]: Mở transaction cho thao tác đổi cấu hình VIP {apps/core-api/src/modules/admin/admin.service.ts:400}
17. admin -> economy [call/di-call]: setVipPlanActive(manager, planId, active) {apps/core-api/src/modules/admin/admin.service.ts:401}
18. admin -> postgres [call/shared-db]: Ghi audit economy.vip_plan.updated cùng manager {apps/core-api/src/modules/admin/admin.service.ts:406}
19. admin -> admin-console [return/http]: Trả VIP plan đã cập nhật

## Invariants

- Admin chỉ gọi EconomyService (getWallet, listTransactions, listCatalogForAdmin) để đọc; không truy cập bảng ledger/wallet trực tiếp, nên Economy vẫn là chủ dữ liệu kinh tế.
- Các endpoint ví, giao dịch dùng @RequireAdminPermission(AdminPermission.RefundTransaction); endpoint catalog và PATCH dùng AdminPermission.ManageConfig, kiểm trước khi vào service.
- Đổi cấu hình IAP/VIP và audit.record dùng chung một manager trong dataSource.transaction, nên không có thay đổi cấu hình nào mà thiếu audit log.

## Failure modes

- when audit.record lỗi trong transaction đổi cấu hình IAP hoặc VIP: Exception thoát khỏi callback dataSource.transaction nên transaction bị rollback, cấu hình không đổi.
- when Admin không có quyền RefundTransaction hoặc ManageConfig: Decorator RequireAdminPermission chặn tại controller, không gọi Economy.

## Endpoints covered

- GET /admin/economy/wallet/:userId [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- GET /admin/economy/users/:userId/transactions [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- GET /admin/config/economy-catalog [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- PATCH /admin/config/iap-products/:productId [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- PATCH /admin/config/vip-plans/:id [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
