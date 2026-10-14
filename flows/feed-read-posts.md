# Đọc feed và bài đăng

id: flow:feed-read-posts
group: Xã hội & nội dung
services: module:feed, module:safety, module:user, module:friend

Người dùng đọc feed công khai, timeline của một tác giả và chi tiết một bài. Feed chỉ sở hữu Post, nên mọi quyết định xem được hay không phải hỏi Safety (block), User (ẩn hồ sơ, hồ sơ tác giả) và Friend (quan hệ bạn bè).

Ba endpoint đọc của Feed: mỗi endpoint hỏi Safety/User/Friend trước hoặc sau khi truy vấn bảng Post, rồi gắn hồ sơ tác giả từ User.

## Participants

- web (client): Web client
- feed (service): Feed
- safety (service): Safety
- user (service): User
- friend (service): Friend
- postgres (infra): PostgreSQL

## Steps

1. web -> feed [call/http]: GET /feed/posts: đọc feed công khai (JWT) (Mọi route REST đều qua JwtAuthGuard toàn cục.) {apps/core-api/src/app/app.module.ts:154}
2. feed -> safety [call/di-call]: Lấy danh sách user đang block/bị block với người xem {apps/core-api/src/modules/feed/feed.service.ts:103}
3. feed -> postgres [call/shared-db]: Query Post public, chưa xoá, loại tác giả bị block, phân trang theo seq (Điều kiện ẩn hồ sơ (hide_profile) được kiểm bằng subquery trên user_privacy_settings ngay trong câu query.) {apps/core-api/src/modules/feed/feed.service.ts:105}
4. feed -> user [call/di-call]: Batch tải hồ sơ tác giả của cả trang {apps/core-api/src/modules/feed/feed.controller.ts:215}
5. feed -> web [return/http]: Trả trang bài đăng kèm hồ sơ tác giả
6. web -> feed [call/http]: GET /feed/users/:userId/posts: timeline một tác giả
7. feed -> safety [call/di-call]: Kiểm tra block hai chiều giữa người xem và tác giả (Nếu bị block, trả trang rỗng thay vì lỗi (chống lộ trạng thái block). Các bước block/ẩn hồ sơ chỉ chạy khi người xem không phải chính tác giả.) {apps/core-api/src/modules/feed/feed.service.ts:150, apps/core-api/src/modules/feed/feed.service.ts:151}
8. feed -> user [call/di-call]: Hỏi tác giả có ẩn hồ sơ không (ẩn thì trả rỗng) {apps/core-api/src/modules/feed/feed.service.ts:153}
9. feed -> friend [call/di-call]: Nếu không phải tác giả: hỏi có là bạn để chọn audience được xem (Tác giả tự xem timeline của mình thì bỏ qua bước này và thấy cả public, friends, only_me; bạn bè thấy public+friends; người lạ chỉ thấy public.) {apps/core-api/src/modules/feed/feed.service.ts:159}
10. feed -> postgres [call/shared-db]: Query Post của tác giả theo audience đã chọn, phân trang seq {apps/core-api/src/modules/feed/feed.service.ts:163}
11. feed -> user [call/di-call]: Batch tải hồ sơ tác giả cho trang kết quả {apps/core-api/src/modules/feed/feed.controller.ts:215}
12. feed -> web [return/http]: Trả timeline kèm hồ sơ tác giả
13. web -> feed [call/http]: GET /feed/posts/:postId: chi tiết một bài
14. feed -> postgres [call/shared-db]: Đọc Post theo id (404 nếu không có hoặc đã xoá) {apps/core-api/src/modules/feed/feed.service.ts:196}
15. feed -> safety [call/di-call]: Kiểm tra block hai chiều với tác giả bài {apps/core-api/src/modules/feed/feed.service.ts:206, apps/core-api/src/modules/feed/feed.service.ts:207}
16. feed -> friend [call/di-call]: Với bài audience friends: hỏi hai người có là bạn {apps/core-api/src/modules/feed/feed.service.ts:211}
17. feed -> user [call/di-call]: Hỏi tác giả có ẩn hồ sơ không (Nếu block, ẩn hồ sơ hoặc audience không cho phép thì cùng ném POST_NOT_FOUND 404.) {apps/core-api/src/modules/feed/feed.service.ts:215}
18. feed -> user [call/di-call]: Tải hồ sơ công khai của tác giả bài {apps/core-api/src/modules/feed/feed.controller.ts:208}
19. feed -> web [return/http]: Trả chi tiết bài kèm hồ sơ tác giả

## Invariants

- Feed không sở hữu quan hệ block, bạn bè hay quyền riêng tư nên phải hỏi Safety, Friend và User bằng lời gọi đồng bộ trước khi trả bài, nếu không sẽ lộ bài của người đã block hoặc bài audience friends.
- Chi tiết bài dùng chung một mã lỗi POST_NOT_FOUND cho bài không tồn tại, đã xoá, bị block, ẩn hồ sơ hoặc audience không cho phép, để người gọi không đoán được lý do thật.
- Hồ sơ tác giả luôn lấy từ User (PublicProfileDto) ở controller, không lưu bản sao trong bảng Post.

## Failure modes

- when Hai người block nhau khi xem timeline của tác giả: Trả trang rỗng với nextCursor null, giống tác giả chưa đăng bài.
- when Cursor phân trang không hợp lệ ở feed hoặc timeline: Ném FeedErrors.CURSOR_INVALID với HTTP 400.
- when Bài không tồn tại, đã xoá, bị block, ẩn hồ sơ hoặc sai audience: Ném POST_NOT_FOUND với HTTP 404.

## Endpoints covered

- GET /feed/posts [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- GET /feed/users/:userId/posts [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- GET /feed/posts/:postId [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
