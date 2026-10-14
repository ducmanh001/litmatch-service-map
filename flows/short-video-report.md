# Báo cáo video vi phạm

id: flow:short-video-report
group: Xã hội & nội dung
services: module:short-video, module:safety

Người xem báo cáo một video ngắn; short-video phải xác nhận video tồn tại và hiển thị được trước, Safety ghi log report append-only và đếm số người báo cáo khác nhau, rồi short-video quyết định tự ẩn video khi vượt ngưỡng.

Luồng POST /videos/:id/report: kiểm tra video, ghi report ở Safety, đếm distinct reporter và tự động gỡ video đang published khi vượt ngưỡng.

## Participants

- web (client): Web client
- short-video (service): Video ngắn
- safety (service): Safety
- postgres (infra): PostgreSQL

## Steps

1. web -> short-video [call/http]: POST /videos/:id/report với reason và description (Mọi route REST đều qua JwtAuthGuard toàn cục.)
2. short-video -> postgres [call/shared-db]: Đọc video theo id, kiểm tra tồn tại và hiển thị với người báo cáo (Video không tồn tại, hoặc chưa published và không phải tác giả, đều trả VIDEO_NOT_FOUND (404) trước khi ghi report.) {apps/core-api/src/modules/short-video/short-video.service.ts:174}
3. short-video -> safety [call/di-call]: Gọi SafetyService.reportVideo(userId, videoId, reason, description) {apps/core-api/src/modules/short-video/short-video.service.ts:490}
4. safety -> postgres [call/shared-db]: INSERT report (unique reporter+video), trùng thì đọc lại; COUNT DISTINCT reporter (Không bọc transaction: insert lỗi unique-violation rồi đọc lại trong cùng transaction sẽ bị Postgres huỷ.) {apps/core-api/src/modules/safety/safety.service.ts:159, apps/core-api/src/modules/safety/safety.service.ts:181}
5. safety -> short-video [return/di-call]: Trả distinctReporterCount, Safety không tự quyết ẩn video
6. short-video -> postgres [call/shared-db]: Nếu count >= ngưỡng: UPDATE status published -> removed (có điều kiện) (Idempotent: thua race hoặc video đã ẩn thì affected = 0, trả false, không lỗi. Dưới ngưỡng thì bước này bị bỏ qua.) {apps/core-api/src/modules/short-video/short-video.service.ts:542, apps/core-api/src/modules/short-video/short-video.service.ts:507}
7. short-video -> web [return/http]: 204 No Content

## Invariants

- short-video phải kiểm tra video tồn tại trước khi gọi Safety vì Safety trung lập và không biết bảng videos; nhờ vậy Safety không ghi report cho video không có thật.
- Safety chỉ trả số distinct reporter còn quyết định ẩn nằm ở short-video, vì chỉ short-video sở hữu vòng đời Video và ngưỡng VIDEO_REPORT_AUTOHIDE_THRESHOLD.
- Ràng buộc unique (reporter, video) ở Safety chặn một người báo cáo lặp lại làm tăng số đếm, nên ngưỡng tự ẩn tính theo người khác nhau.

## Failure modes

- when Video không tồn tại hoặc chưa published và người gọi không phải tác giả: short-video ném VIDEO_NOT_FOUND (404), Safety không được gọi, không ghi report.
- when Cùng người báo cáo cùng video nhiều lần: Insert vi phạm unique, Safety đọc lại report cũ và không tăng số distinct reporter.
- when Hai request cùng vượt ngưỡng đồng thời: UPDATE có điều kiện status = published chỉ thắng một lần, request thua nhận affected = 0 và không ném lỗi.

## Endpoints covered

- POST /videos/:id/report [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
