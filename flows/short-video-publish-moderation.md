# Đăng video ngắn và duyệt bởi admin

id: flow:short-video-publish-moderation
group: Xã hội & nội dung
services: module:short-video, module:admin, module:user

Người dùng xin URL upload, upload video rồi finalize; tuỳ VIDEO_MODERATION_MODE video sang pending_review hoặc published. Admin duyệt qua AdminModule, nên short-video (sở hữu Video) và admin (audit log) phải cùng một transaction.

Vòng đời Video từ upload-intent, finalize đến admin duyệt, và ranh giới transaction giữa admin và short-video.

## Participants

- web (client): Web client
- admin-console (client): Admin console
- short-video (service): Short-video
- admin (service): Admin
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> short-video [call/http]: POST /videos/upload-intent kèm Idempotency-Key và caption
2. short-video -> postgres [call/shared-db]: Lưu Video trạng thái uploading với storageKey đã sinh (Nếu unique violation (replay) thì đọc lại video gốc theo idempotencyKey và dùng lại storageKey cũ.) {apps/core-api/src/modules/short-video/short-video.service.ts:81}
3. short-video -> web [return/http]: Trả video và uploadUrl (presigned) để upload thẳng lên storage (Presigned uploadUrl được lấy từ VideoStoragePort.issueUploadUrl SAU khi lưu Video (dùng storageKey của video gốc nếu replay); body video không đi qua core-api.)
4. web -> short-video [call/http]: POST /videos/:id/finalize sau khi upload xong
5. short-video -> postgres [call/shared-db]: Đọc video và kiểm tra tác giả là chính người gọi {apps/core-api/src/modules/short-video/short-video.service.ts:516}
6. short-video -> postgres [call/shared-db]: UPDATE có điều kiện uploading sang processing (thua race thì 409) (transition là UPDATE ... WHERE id AND status=uploading; chỉ khi thắng (affected>0) mới tiếp tục sang transcode.) {apps/core-api/src/modules/short-video/short-video.service.ts:541}
7. short-video -> postgres [call/shared-db]: Sau transcode: cập nhật pending_review (mode pre) hoặc published kèm playbackUrl (Gọi VideoTranscodePort.transcode (không vẽ riêng) rồi UPDATE có điều kiện status=processing; VIDEO_MODERATION_MODE='pre' thì pending_review, ngược lại published. Mọi lỗi trong try (transcode hoặc update) làm catch chuyển processing sang failed; kết quả update không được kiểm.) {apps/core-api/src/modules/short-video/short-video.service.ts:150}
8. short-video -> user [call/di-call]: Lấy hồ sơ tác giả để dựng PublicProfileDto {apps/core-api/src/modules/short-video/short-video.controller.ts:209}
9. short-video -> web [return/http]: Trả Video với trạng thái mới (pending_review hoặc published) (Route yêu cầu quyền AdminPermission.ResolveReports (RequireAdminPermission), ngoài JWT guard toàn cục.)
10. admin-console -> admin [call/http]: POST /admin/videos/:id/approve
11. admin -> postgres [call/shared-db]: Mở transaction cho duyệt video {apps/core-api/src/modules/admin/admin.service.ts:161}
12. admin -> short-video [call/di-call]: adminApprove(videoId, manager): chuyển pending_review sang published {apps/core-api/src/modules/admin/admin.service.ts:162}
13. short-video -> postgres [call/shared-db]: UPDATE có điều kiện trên manager của admin (cùng transaction) {apps/core-api/src/modules/short-video/short-video.service.ts:541}
14. admin -> postgres [call/shared-db]: Ghi audit log video.approved cùng manager {apps/core-api/src/modules/admin/admin.service.ts:163}
15. admin -> admin-console [return/http]: Trả Video đã duyệt sau khi transaction commit

## Invariants

- Chuyển trạng thái Video dùng UPDATE có điều kiện theo status hiện tại (from), nên hai request đua nhau chỉ một bên thắng; bên thua nhận affected = 0 và finalize trả 409.
- AdminModule mở transaction và truyền manager vào short-video.adminApprove, nên đổi trạng thái Video và audit log video.approved cùng commit hoặc cùng rollback; short-video là bên sở hữu dữ liệu Video.

## Failure modes

- when Transcode ném lỗi trong finalize: catch chuyển video từ processing sang failed rồi vẫn trả về video đọc lại từ DB.
- when Finalize khi video không còn ở trạng thái uploading: transition trả false, service ném VIDEO_INVALID_TRANSITION (409).
- when Upload-intent bị replay cùng Idempotency-Key nhưng khác caption: ném VIDEO_UPLOAD_IDEMPOTENCY_CONFLICT (409).
- when Admin duyệt video không còn pending_review: adminApprove dùng transitionOrThrow: ném VIDEO_INVALID_TRANSITION (409) hoặc 404 nếu không có video; exception làm transaction rollback nên audit log video.approved không được ghi.

## Endpoints covered

- POST /videos/upload-intent [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- POST /videos/:id/finalize [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- POST /admin/videos/:id/approve [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
