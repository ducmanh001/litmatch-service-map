# Admin xử lý báo cáo vi phạm

id: flow:admin-report-resolution
group: Vận hành (admin)
services: module:admin, module:safety

Admin xem hàng đợi báo cáo rồi đánh dấu đã xử lý hoặc bác bỏ. AdminModule kiểm quyền và điều phối, SafetyModule sở hữu bảng report, và việc đổi trạng thái phải ghi cùng transaction với audit log.

Admin liệt kê báo cáo qua Safety, rồi đổi trạng thái báo cáo và ghi audit log trong cùng một transaction.

## Participants

- admin-console (client): Admin console
- admin (service): Admin
- safety (service): Safety
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/reports lọc theo status, phân trang offset (Route yêu cầu quyền ResolveReports qua AdminPermissionGuard.) {apps/core-api/src/modules/admin/admin.controller.ts:170}
2. admin -> safety [call/di-call]: findReportsPage(filter, limit, offset) {apps/core-api/src/modules/admin/admin.service.ts:112}
3. safety -> postgres [call/shared-db]: Truy vấn report, lọc status, sắp theo createdAt giảm dần {apps/core-api/src/modules/safety/safety.service.ts:342}
4. safety -> admin [return/di-call]: Trả trang báo cáo gồm items và total
5. admin -> admin-console [return/http]: Trả AdminReportsPageDto cho hàng đợi kiểm duyệt
6. admin-console -> admin [call/http]: POST /admin/reports/:id/resolve hoặc /dismiss {apps/core-api/src/modules/admin/admin.controller.ts:185}
7. admin -> postgres [call/shared-db]: Mở transaction cho setReportStatus (resolve dùng ReportStatus.Resolved, dismiss dùng ReportStatus.Dismissed.) {apps/core-api/src/modules/admin/admin.service.ts:128}
8. admin -> safety [call/di-call]: setReportStatus(manager, reportId, status) {apps/core-api/src/modules/admin/admin.service.ts:129}
9. safety -> postgres [call/shared-db]: Đọc report theo id rồi lưu status mới (Không thấy report thì ném 404 REPORT_NOT_FOUND; chỉ status bị đổi.) {apps/core-api/src/modules/safety/safety.service.ts:360, apps/core-api/src/modules/safety/safety.service.ts:369}
10. admin -> postgres [call/shared-db]: Ghi audit log report.<status> bằng cùng manager {apps/core-api/src/modules/admin/admin.service.ts:134}
11. admin -> admin-console [return/http]: Trả AdminReportDto sau khi transaction hoàn tất

## Invariants

- Safety sở hữu bảng report nên Admin không tự ghi; Admin chỉ truyền manager để thay đổi trạng thái và audit log nằm trong cùng transaction.
- Ghi audit sau khi Safety đổi status trong cùng callback, nên nếu một bước ném lỗi thì không có trạng thái đổi mà thiếu audit.

## Failure modes

- when Report id không tồn tại: SafetyService.setReportStatus ném DomainException 404 REPORT_NOT_FOUND, callback transaction dừng và không ghi audit.
- when Admin thiếu quyền ResolveReports: AdminPermissionGuard chặn trước khi vào service, không có truy vấn nào chạy.

## Endpoints covered

- GET /admin/reports [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/reports/:id/resolve [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/reports/:id/dismiss [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
