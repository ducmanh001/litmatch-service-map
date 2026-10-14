# Xoá bài và bình luận

id: flow:feed-delete-post
group: Xã hội & nội dung
services: module:feed, module:safety, module:friend, module:user

Người dùng xoá mềm bài viết hoặc bình luận của chính mình trong feed; feed phải hỏi safety, friend, user về quyền xem bài trước khi đối chiếu quyền sở hữu, rồi ghi deletedAt vào PostgreSQL.

Hai endpoint xoá mềm của feed: bài viết đi qua guard getPostOrThrow (safety, friend, user) rồi update deletedAt; bình luận kiểm tra tác giả rồi cập nhật deletedAt và giảm commentCount trong một transaction.

## Participants

- web (client): Web client
- feed (service): Feed
- safety (service): Safety
- friend (service): Friend
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> feed [call/http]: DELETE /feed/posts/:postId (JWT, trả 204 khi thành công) (Mọi route REST đều qua JwtAuthGuard toàn cục trừ route @Public.) {apps/core-api/src/modules/feed/feed.controller.ts:128}
2. feed -> postgres [call/shared-db]: getPostOrThrow: đọc Post theo id, 404 nếu không có hoặc đã xoá {apps/core-api/src/modules/feed/feed.service.ts:196}
3. feed -> safety [call/di-call]: isBlocked hai chiều giữa người gọi và tác giả bài (Chỉ chạy khi post.authorUserId khác user.userId (đọc code dòng 204); chủ bài tự xoá thì bỏ qua các bước 3-5.) {apps/core-api/src/modules/feed/feed.service.ts:206}
4. feed -> friend [call/di-call]: areFriends khi bài có audience Friends {apps/core-api/src/modules/feed/feed.service.ts:211}
5. feed -> user [call/di-call]: PrivacySettingsService.isHidden của tác giả {apps/core-api/src/modules/feed/feed.service.ts:215}
6. feed -> postgres [call/shared-db]: Chỉ tác giả được xoá (khác thì 403); update Post.deletedAt = now {apps/core-api/src/modules/feed/feed.service.ts:237}
7. feed -> web [return/http]: 204 No Content cho việc xoá bài
8. web -> feed [call/http]: DELETE /feed/comments/:commentId (JWT, trả 204 khi thành công) {apps/core-api/src/modules/feed/feed.controller.ts:165}
9. feed -> postgres [call/shared-db]: Đọc Comment; 404 nếu không có/đã xoá, 403 nếu không phải tác giả {apps/core-api/src/modules/feed/feed.service.ts:314}
10. feed -> postgres [call/shared-db]: Trong transaction: update Comment.deletedAt khi còn NULL {apps/core-api/src/modules/feed/feed.service.ts:329}
11. feed -> postgres [call/shared-db]: Cũng trong transaction: giảm Post.commentCount nếu update có affected = 1 {apps/core-api/src/modules/feed/feed.service.ts:339}
12. feed -> web [return/http]: 204 No Content cho việc xoá bình luận

## Invariants

- Xoá bài đi qua getPostOrThrow, guard trung tâm của feed, nên block/privacy/audience luôn do safety, friend, user quyết định, feed không tự giữ các quan hệ này.
- Việc kiểm tra tác giả của comment nằm ngoài transaction, nên update chỉ áp dụng khi deletedAt còn NULL và commentCount chỉ giảm khi affected = 1, tránh giảm đếm hai lần khi xoá trùng.
- Xoá mềm chỉ ghi deletedAt, không xoá dòng; getPostOrThrow và deleteComment coi bản ghi có deletedAt là không tồn tại (404), còn các đường đọc khác Suy luận: cũng lọc deletedAt.

## Failure modes

- when Bài không tồn tại hoặc đã xoá mềm: getPostOrThrow ném POST_NOT_FOUND với HTTP 404, không ghi gì.
- when Người gọi không phải tác giả bài: Nếu bị block/hidden/audience không cho xem thì 404, còn lại 403 'Bài viết không thuộc về bạn'; không update.
- when Comment không thuộc về người gọi: deleteComment ném HTTP 403 trước khi mở transaction.
- when Hai request xoá cùng một comment đua nhau: Chỉ request làm update affected = 1 mới giảm commentCount, request còn lại return mà không giảm.

## Endpoints covered

- DELETE /feed/posts/:postId [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- DELETE /feed/comments/:commentId [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
