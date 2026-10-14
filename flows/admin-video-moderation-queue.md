# Admin duyệt hàng chờ video

id: flow:admin-video-moderation-queue
group: Vận hành (admin)
services: module:admin, module:short-video

Moderator xem video chờ duyệt và video đã đăng, rồi từ chối hoặc gỡ video. AdminModule kiểm quyền và ghi audit log, còn Short Video là nơi sở hữu bảng videos và quy tắc chuyển trạng thái, nên hai module phải phối hợp trong cùng một transaction.

Admin-console gọi AdminModule, AdminModule gọi ShortVideoService để đọc hàng chờ hoặc đổi trạng thái video, và ghi audit log cùng transaction.

## Participants

- admin-console (client): Admin console
- admin (service): Admin (RBAC, audit)
- short-video (service): Short Video
- postgres (infra): PostgreSQL

## Steps

1. admin-console -> admin [call/http]: GET /admin/videos/pending: xem video chờ duyệt (cần quyền ResolveReports) (Route yêu cầu @RequireAdminPermission(AdminPermission.ResolveReports); cả hai route danh sách và hai route hành động dùng cùng quyền này.) {apps/core-api/src/modules/admin/admin.controller.ts:212}
2. admin -> short-video [call/di-call]: listPendingReview(query): lấy trang video chờ duyệt {apps/core-api/src/modules/admin/admin.service.ts:151}
3. short-video -> postgres [call/shared-db]: Đọc videos có status = pending_review, phân trang cursor {apps/core-api/src/modules/short-video/short-video.service.ts:431}
4. admin -> admin-console [return/http]: Trả trang video chờ duyệt (AdminVideosPageDto)
5. admin-console -> admin [call/http]: GET /admin/videos/published: xem video đang published {apps/core-api/src/modules/admin/admin.controller.ts:228}
6. admin -> short-video [call/di-call]: listPublished({ ...query, sort: 'recent' }): lấy video published mới nhất {apps/core-api/src/modules/admin/admin.service.ts:157}
7. admin-console -> admin [call/http]: POST /admin/videos/:id/reject: từ chối video chờ duyệt {apps/core-api/src/modules/admin/admin.controller.ts:260}
8. admin -> postgres [call/shared-db]: Mở transaction rejectVideo {apps/core-api/src/modules/admin/admin.service.ts:177}
9. admin -> short-video [call/di-call]: adminReject(videoId, manager): pending_review -> rejected (UPDATE có điều kiện status = pending_review; 0 dòng bị ảnh hưởng thì ném VIDEO_INVALID_TRANSITION (409) hoặc 404 nếu không có video, transaction rollback.) {apps/core-api/src/modules/admin/admin.service.ts:178}
10. short-video -> postgres [call/shared-db]: UPDATE videos SET status = rejected WHERE id và status = pending_review {apps/core-api/src/modules/short-video/short-video.service.ts:542}
11. admin -> postgres [call/shared-db]: auditLogService.record: ghi audit video.rejected cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:179}
12. admin -> admin-console [return/http]: Trả AdminVideoDto của video đã bị từ chối
13. admin-console -> admin [call/http]: POST /admin/videos/:id/remove: gỡ thủ công video đang published {apps/core-api/src/modules/admin/admin.controller.ts:276}
14. admin -> postgres [call/shared-db]: Mở transaction removeVideo {apps/core-api/src/modules/admin/admin.service.ts:194}
15. admin -> short-video [call/di-call]: adminRemove(videoId, manager): published -> removed {apps/core-api/src/modules/admin/admin.service.ts:195}
16. short-video -> postgres [call/shared-db]: UPDATE videos SET status = removed WHERE id và status = published {apps/core-api/src/modules/short-video/short-video.service.ts:542}
17. admin -> postgres [call/shared-db]: auditLogService.record: ghi audit video.removed cùng transaction {apps/core-api/src/modules/admin/admin.service.ts:196}
18. admin -> admin-console [return/http]: Trả AdminVideoDto của video đã bị gỡ

## Invariants

- AdminModule không sở hữu bảng videos: nó phải gọi ShortVideoService để đổi trạng thái, nhưng truyền manager của mình vào để việc đổi trạng thái và audit log nằm trong cùng một transaction.
- Chuyển trạng thái dùng UPDATE có điều kiện trên trạng thái nguồn (reject chỉ từ pending_review, remove chỉ từ published), nên hai moderator thao tác cùng video thì chỉ một bên thắng, bên kia nhận lỗi rõ ràng thay vì ghi đè.

## Failure modes

- when Video không còn ở trạng thái nguồn (đã được duyệt, từ chối hoặc gỡ bởi người khác): transitionOrThrow ném VIDEO_INVALID_TRANSITION với HTTP 409; transaction rollback, không có audit log.
- when Không tìm thấy video theo id: Ném lỗi 404 'Không tìm thấy video'; không ghi audit.
- when Ghi audit log lỗi: Exception thoát khỏi callback transaction nên việc đổi trạng thái video cũng rollback.
- when Moderator lật sang trang 2 của hàng chờ pending_review bằng cursor: Suy luận: listPendingReview sắp createdAt tăng dần nhưng điều kiện cursor là (createdAt, id) < cursor nên trang sau có thể lặp lại hoặc bỏ sót video cũ; cần kiểm lại hướng so sánh.

## Endpoints covered

- GET /admin/videos/pending [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- GET /admin/videos/published [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/videos/:id/reject [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- POST /admin/videos/:id/remove [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
