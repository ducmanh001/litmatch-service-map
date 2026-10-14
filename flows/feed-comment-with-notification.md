# Bình luận bài kèm thông báo

id: flow:feed-comment-with-notification
group: Xã hội & nội dung
services: module:feed, module:safety, module:friend, module:user, module:notification

Người dùng bình luận lên bài đăng: feed phải hỏi safety (block hai chiều), friend (audience bạn bè) và user (ẩn hồ sơ) trước khi ghi, rồi lưu comment cùng notification cho tác giả trong một transaction và đẩy push sau commit.

Feed kiểm tra quyền xem bài với safety/friend/user, ghi comment và notification trong một transaction, rồi đẩy push best-effort.

## Participants

- web (client): Web client
- feed (service): Feed
- safety (service): Safety
- friend (service): Friend
- user (service): User
- notification (service): Notification
- postgres (infra): PostgreSQL

## Steps

1. web -> feed [call/http]: POST /feed/posts/:postId/comments gửi nội dung bình luận (Route được bảo vệ bởi JwtAuthGuard toàn cục; guest bị chặn bởi assertNotGuest trong service.)
2. feed -> postgres [call/shared-db]: Đọc Post theo id, 404 nếu không có hoặc đã xoá mềm {apps/core-api/src/modules/feed/feed.service.ts:196}
3. feed -> safety [call/di-call]: Hỏi isBlocked hai chiều giữa người bình luận và tác giả {apps/core-api/src/modules/feed/feed.service.ts:206, apps/core-api/src/modules/feed/feed.service.ts:207}
4. feed -> friend [call/di-call]: Bài audience friends: hỏi areFriends(người bình luận, tác giả) {apps/core-api/src/modules/feed/feed.service.ts:211}
5. feed -> user [call/di-call]: Hỏi PrivacySettingsService.isHidden(tác giả) (Nếu bị block, hidden hoặc audience không cho phép thì ném POST_NOT_FOUND (404) để không lộ lý do.) {apps/core-api/src/modules/feed/feed.service.ts:215}
6. feed -> postgres [call/shared-db]: Transaction: lưu Comment và tăng Post.commentCount {apps/core-api/src/modules/feed/feed.service.ts:249}
7. feed -> notification [call/di-call]: createWithManager thông báo PostCommented cho tác giả (Chỉ gọi khi người bình luận khác tác giả bài.) {apps/core-api/src/modules/feed/feed.service.ts:260}
8. notification -> postgres [call/shared-db]: Lưu hàng notifications bằng manager của transaction feed {apps/core-api/src/modules/notification/notification.service.ts:59}
9. feed -> notification [async/di-call]: Sau commit: sendPush cho thông báo vừa tạo (sendPush bọc try/catch, lỗi push chỉ ghi log warn, không làm hỏng bình luận.) {apps/core-api/src/modules/feed/feed.service.ts:275}
10. feed -> user [call/di-call]: Controller lấy hồ sơ tác giả comment để dựng PublicProfileDto {apps/core-api/src/modules/feed/feed.controller.ts:208}
11. feed -> web [return/http]: Trả comment vừa tạo kèm thông tin tác giả
12. web -> feed [call/http]: GET /feed/posts/:postId/comments phân trang theo cursor seq
13. feed -> safety [call/di-call]: Tái dùng getPostOrThrow: lại kiểm block, friend, hidden (Cùng guard như lúc tạo comment (isBlocked, areFriends, isHidden) nên người bị block không đọc được bình luận.) {apps/core-api/src/modules/feed/feed.service.ts:284}
14. feed -> postgres [call/shared-db]: Query Comment chưa xoá, seq > cursor, sắp xếp tăng dần {apps/core-api/src/modules/feed/feed.service.ts:299}
15. feed -> user [call/di-call]: Controller gọi findByIds lấy hồ sơ các tác giả comment {apps/core-api/src/modules/feed/feed.controller.ts:215}
16. feed -> web [return/http]: Trả trang bình luận kèm nextCursor

## Invariants

- Phải kiểm block hai chiều, audience bạn bè và hidden TRƯỚC khi ghi: feed không sở hữu quan hệ này nên bắt buộc hỏi safety, friend, user và trả 404 thay vì 403 để không lộ lý do.
- Comment, tăng commentCount và (khi người bình luận khác tác giả bài) hàng notification nằm cùng một transaction, notification dùng manager của feed, nên không có comment của người khác mà thiếu thông báo hoặc ngược lại; tự bình luận bài mình thì không tạo notification.
- sendPush chạy sau khi transaction commit và nuốt lỗi, vì push là hiệu ứng phụ: thông báo in-app trong DB mới là nguồn sự thật.

## Failure modes

- when Người bình luận và tác giả block nhau (một trong hai chiều), hoặc tác giả ẩn hồ sơ, hoặc bài friends mà không phải bạn: getPostOrThrow ném POST_NOT_FOUND 404, không ghi comment hay notification.
- when Lưu notification hoặc tăng commentCount lỗi trong transaction: Transaction rollback, comment không được tạo và không có push.
- when Push provider hoặc web push lỗi sau commit: sendPush bắt lỗi và ghi log warn; comment và thông báo in-app vẫn được giữ.

## Endpoints covered

- POST /feed/posts/:postId/comments [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- GET /feed/posts/:postId/comments [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
