# Reaction bài kèm thông báo

id: flow:feed-reaction-with-notification
group: Xã hội & nội dung
services: module:feed, module:safety, module:friend, module:user, module:notification

Người dùng thả tim, bỏ tim và xem trạng thái reaction của một bài đăng. Feed sở hữu Reaction nhưng quyền xem bài phụ thuộc safety (block), friend (audience bạn bè) và user (ẩn hồ sơ), còn thông báo cho tác giả phải ghi qua notification trong cùng transaction.

Luồng like/unlike/xem reaction: feed hỏi safety, friend, user rồi ghi Reaction và Notification trong một transaction, push gửi sau best-effort.

## Participants

- web (client): Web client
- feed (service): Feed
- safety (service): Safety
- friend (service): Friend
- user (service): User
- notification (service): Notification
- postgres (infra): PostgreSQL

## Steps

1. web -> feed [call/http]: POST /feed/posts/:postId/reactions (thả tim) (Mọi route REST đều qua JwtAuthGuard toàn cục; like chặn tài khoản guest (assertNotGuest).)
2. feed -> postgres [call/shared-db]: getPostOrThrow: đọc Post theo id {apps/core-api/src/modules/feed/feed.service.ts:196}
3. feed -> safety [call/di-call]: isBlocked hai chiều giữa người xem và tác giả (Chỉ chạy khi người xem không phải tác giả.) {apps/core-api/src/modules/feed/feed.service.ts:206}
4. feed -> friend [call/di-call]: areFriends khi audience của bài là friends {apps/core-api/src/modules/feed/feed.service.ts:211}
5. feed -> user [call/di-call]: isHidden: tác giả có ẩn hồ sơ không (Nếu bị block, ẩn hoặc audience không cho phép thì ném POST_NOT_FOUND (không lộ lý do).) {apps/core-api/src/modules/feed/feed.service.ts:215}
6. feed -> postgres [call/shared-db]: Transaction: lưu Reaction và tăng Post.likeCount (Unique trên Reaction chặn like trùng; vi phạm unique được bắt và trả liked=true không tăng đếm.) {apps/core-api/src/modules/feed/feed.service.ts:353}
7. feed -> notification [call/di-call]: createWithManager: thông báo PostLiked cho tác giả (Bỏ qua khi tự like bài của mình.) {apps/core-api/src/modules/feed/feed.service.ts:360}
8. notification -> postgres [call/shared-db]: Ghi dòng notifications bằng manager của transaction {apps/core-api/src/modules/notification/notification.service.ts:60}
9. feed -> notification [async/di-call]: sendPush sau khi commit (best-effort) (sendPush bắt mọi lỗi push và chỉ ghi log, không làm hỏng like.) {apps/core-api/src/modules/feed/feed.service.ts:370}
10. feed -> web [return/http]: Trả { liked: true, likeCount } (likeCount không đọc lại sau transaction mà tính từ bản Post đã đọc trước đó (likeCount + 1), nên có thể lệch nhẹ khi nhiều request đồng thời.)
11. web -> feed [call/http]: DELETE /feed/posts/:postId/reactions (bỏ tim)
12. feed -> safety [call/di-call]: Kiểm tra lại quyền xem bài như lúc like (Cùng getPostOrThrow: đọc Post, rồi safety, friend, user.) {apps/core-api/src/modules/feed/feed.service.ts:207}
13. feed -> postgres [call/shared-db]: Transaction: xoá Reaction, giảm likeCount nếu có xoá (Không gửi thông báo khi bỏ tim.) {apps/core-api/src/modules/feed/feed.service.ts:384}
14. feed -> web [return/http]: Trả { liked: false, likeCount } (Tương tự: likeCount suy từ bản Post đã đọc trước (trừ 1 nếu có xoá được Reaction), không đọc lại.)
15. web -> feed [call/http]: GET /feed/posts/:postId/reactions (trạng thái) (Cũng qua getPostOrThrow với safety, friend, user như trên.)
16. feed -> postgres [call/shared-db]: exists(Reaction) của người xem trên bài {apps/core-api/src/modules/feed/feed.service.ts:405}
17. feed -> web [return/http]: Trả { liked, likeCount }

## Invariants

- Feed phải hỏi safety, friend và user trước khi ghi vì Feed không sở hữu block, bạn bè hay privacy; getPostOrThrow là guard chung nên like, unlike và xem trạng thái đều cùng quyết định xem được bài.
- Reaction, Post.likeCount và Notification được ghi trong cùng một transaction để thông báo không tồn tại nếu like rollback; push chỉ gửi sau khi commit vì sendPush best-effort và không throw.

## Failure modes

- when Người dùng like lặp hoặc hai request đua nhau: Unique của Reaction báo vi phạm, transaction rollback, API trả liked=true với likeCount cũ, không có thông báo trùng.
- when Push provider hoặc web push lỗi: sendPush bắt lỗi và ghi log; like và thông báo in-app vẫn được giữ.
- when Người xem bị block, tác giả ẩn hồ sơ hoặc audience không cho phép: getPostOrThrow ném POST_NOT_FOUND 404, không ghi gì.

## Endpoints covered

- POST /feed/posts/:postId/reactions [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- DELETE /feed/posts/:postId/reactions [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- GET /feed/posts/:postId/reactions [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
