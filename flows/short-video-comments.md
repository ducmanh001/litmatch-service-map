# Bình luận video ngắn

id: flow:short-video-comments
group: Xã hội & nội dung
services: module:short-video, module:user

Người dùng thêm, xem và xoá comment của video ngắn. short-video sở hữu video và comment (kèm commentCount), nhưng hồ sơ tác giả thuộc module user nên controller phải gọi UserService để ghép PublicProfileDto vào kết quả.

Ba endpoint comment của short-video: ghi/đọc/xoá trong PostgreSQL và lấy hồ sơ tác giả từ module user.

## Participants

- web (client): Web client
- short-video (service): short-video
- user (service): user
- postgres (infra): PostgreSQL

## Steps

1. web -> short-video [call/http]: POST /videos/:id/comments gửi nội dung bình luận
2. short-video -> postgres [call/shared-db]: getVideoOrThrow: đọc Video, kiểm tra quyền nhìn thấy {apps/core-api/src/modules/short-video/short-video.service.ts:174}
3. short-video -> postgres [call/shared-db]: Transaction: lưu VideoComment và tăng Video.commentCount {apps/core-api/src/modules/short-video/short-video.service.ts:350}
4. short-video -> user [call/di-call]: getAuthor: UserService.getByIdOrThrow lấy hồ sơ tác giả {apps/core-api/src/modules/short-video/short-video.controller.ts:209}
5. short-video -> web [return/http]: Trả 201 VideoCommentDto kèm hồ sơ tác giả
6. web -> short-video [call/http]: GET /videos/:id/comments với cursor phân trang
7. short-video -> postgres [call/shared-db]: getVideoOrThrow: đọc Video trước khi liệt kê {apps/core-api/src/modules/short-video/short-video.service.ts:174}
8. short-video -> postgres [call/shared-db]: Truy vấn comment chưa xoá, sắp theo seq giảm dần {apps/core-api/src/modules/short-video/short-video.service.ts:370}
9. short-video -> user [call/di-call]: getAuthors: UserService.findByIds tải hồ sơ cả trang một lần {apps/core-api/src/modules/short-video/short-video.controller.ts:216}
10. short-video -> web [return/http]: Trả trang comment kèm hồ sơ tác giả
11. web -> short-video [call/http]: DELETE /videos/:id/comments/:commentId
12. short-video -> postgres [call/shared-db]: Đọc Video và VideoComment để kiểm tra tồn tại và quyền xoá {apps/core-api/src/modules/short-video/short-video.service.ts:391}
13. short-video -> postgres [call/shared-db]: Transaction: đặt deletedAt cho comment và giảm commentCount {apps/core-api/src/modules/short-video/short-video.service.ts:412}
14. short-video -> web [return/http]: Trả 204 No Content

## Invariants

- Lưu comment và tăng Video.commentCount nằm trong cùng một dataSource.transaction (và xoá mềm kèm giảm commentCount cũng vậy), nên số đếm không lệch với số comment.
- getVideoOrThrow luôn chạy trước mọi thao tác comment: chỉ tác giả hoặc video đã published mới được thao tác, và các trường hợp không hợp lệ trả cùng một mã lỗi.
- Hồ sơ tác giả thuộc module user nên short-video lấy qua UserService ở controller; danh sách dùng findByIds một lần cho cả trang thay vì gọi từng người.
- Xoá comment chỉ cho tác giả comment hoặc tác giả video, kiểm tra trước khi mở transaction.

## Failure modes

- when Video không tồn tại, bị gỡ hoặc chưa published với người không phải tác giả: getVideoOrThrow từ chối, comment không được tạo/liệt kê/xoá
- when Comment không tồn tại hoặc đã có deletedAt khi xoá: Ném DomainException 404 'Không tìm thấy comment', không mở transaction
- when Người xoá không phải tác giả comment hay tác giả video: Ném DomainException 403, không có thay đổi dữ liệu
- when Lỗi trong transaction tạo hoặc xoá comment: Cả lưu/cập nhật comment và thay đổi commentCount cùng bị rollback

## Endpoints covered

- POST /videos/:id/comments [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- GET /videos/:id/comments [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- DELETE /videos/:id/comments/:commentId [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
