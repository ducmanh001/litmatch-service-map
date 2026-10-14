# Duyệt video ngắn, xem chi tiết, view và reaction

id: flow:short-video-browse-and-detail
group: Xã hội & nội dung
services: module:short-video, module:friend, module:user

Người dùng duyệt danh sách video published (recent hoặc ranked, hoặc tab following), mở chi tiết, ghi watch-time và thả/bỏ tim. short-video sở hữu dữ liệu video nhưng phải gọi friend để lấy danh sách bạn bè và user để lấy hồ sơ tác giả.

short-video đọc/ghi PostgreSQL của chính nó, mượn friend và user để lọc theo bạn bè và dựng hồ sơ tác giả.

## Participants

- web (client): Web client
- short-video (service): short-video
- friend (service): friend
- user (service): user
- postgres (infra): PostgreSQL

## Steps

1. web -> short-video [call/http]: GET /videos?sort=recent|ranked&feed=following
2. short-video -> friend [call/di-call]: Chỉ khi feed=following: lấy danh sách id bạn bè (Nếu không có bạn nào thì trả trang rỗng ngay, không truy vấn video.) {apps/core-api/src/modules/short-video/short-video.service.ts:206}
3. short-video -> postgres [call/shared-db]: Truy vấn video published, sắp theo createdAt hoặc rankScore, phân trang cursor {apps/core-api/src/modules/short-video/short-video.service.ts:213}
4. short-video -> user [call/di-call]: Nạp hàng loạt hồ sơ tác giả của cả trang {apps/core-api/src/modules/short-video/short-video.controller.ts:216}
5. short-video -> web [return/http]: Trang video kèm tác giả và nextCursor
6. web -> short-video [call/http]: GET /videos/:id
7. short-video -> postgres [call/shared-db]: Đọc video; tác giả thấy mọi status, người khác chỉ published {apps/core-api/src/modules/short-video/short-video.service.ts:174}
8. short-video -> user [call/di-call]: Lấy hồ sơ công khai của tác giả video {apps/core-api/src/modules/short-video/short-video.controller.ts:209}
9. short-video -> web [return/http]: Chi tiết video kèm tác giả
10. web -> short-video [call/http]: POST /videos/:id/views với watchTimeMs
11. short-video -> postgres [call/shared-db]: Kiểm tra video hiển thị được (getVideoOrThrow) {apps/core-api/src/modules/short-video/short-video.service.ts:174}
12. short-video -> postgres [call/shared-db]: Upsert VideoView, cập nhật watchTime, tăng viewCount một lần khi qualified (Self-view bị bỏ qua trước khi mở transaction.) {apps/core-api/src/modules/short-video/short-video.service.ts:268}
13. short-video -> web [return/http]: 204 No Content
14. web -> short-video [call/http]: POST /videos/:id/reactions (thả tim)
15. short-video -> postgres [call/shared-db]: Lưu VideoReaction và tăng likeCount; trùng unique thì coi như đã tim {apps/core-api/src/modules/short-video/short-video.service.ts:310}
16. short-video -> web [return/http]: liked=true cùng likeCount
17. web -> short-video [call/http]: DELETE /videos/:id/reactions (bỏ tim)
18. short-video -> postgres [call/shared-db]: Xoá VideoReaction, giảm likeCount nếu có bản ghi bị xoá {apps/core-api/src/modules/short-video/short-video.service.ts:328}
19. short-video -> web [return/http]: liked=false cùng likeCount

## Invariants

- getVideoOrThrow chạy trước mọi thao tác view/reaction: video không tồn tại, bị removed/rejected hoặc chưa published (với người không phải tác giả) trả cùng mã VIDEO_NOT_FOUND.
- Bản ghi VideoView/VideoReaction và bộ đếm viewCount/likeCount nằm trong cùng một transaction để bộ đếm không lệch với bản ghi.
- Hồ sơ tác giả luôn do module user cung cấp và danh sách bạn bè do module friend cung cấp; short-video không đọc trực tiếp bảng của hai module đó.

## Failure modes

- when Video không tồn tại hoặc không hiển thị với người gọi: getVideoOrThrow ném VIDEO_NOT_FOUND (404), không có ghi DB nào xảy ra.
- when Thả tim trùng (unique violation): Giao dịch bị rollback, API vẫn trả liked=true với likeCount hiện tại (idempotent).
- when Ghi view đồng thời lần đầu bị unique violation: Code bắt lỗi unique, đọc lại VideoView rồi cập nhật; lỗi khác được ném ra. Suy luận: Postgres huỷ transaction sau lỗi unique nên lệnh đọc lại trong cùng transaction có thể vẫn lỗi.
- when feed=following nhưng người dùng chưa có bạn: Trả trang rỗng với nextCursor null, không lỗi.

## Endpoints covered

- GET /videos [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- GET /videos/:id [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- POST /videos/:id/views [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- POST /videos/:id/reactions [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- DELETE /videos/:id/reactions [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
