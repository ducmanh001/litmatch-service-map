# Danh sách follower/following và danh sách bạn bè

id: flow:friend-follow-lists-and-connections
group: Xã hội & nội dung
services: module:friend, module:safety, module:user

Trang hồ sơ và trang Bạn bè đọc số follow, danh sách follower/following theo cursor và danh sách bạn bè (ghép đôi hoặc follow 2 chiều). Friend giữ bảng follow/friendship nên tự truy vấn Postgres; hồ sơ công khai lấy từ user, còn quy tắc block lấy từ safety. Block 2 chiều với người xem làm hồ sơ biến thành 404 và lọc người bị chặn khỏi từng dòng.

Bốn lần đọc mới của friend: đều kiểm block qua safety, đọc follow/friendship trong Postgres rồi ghép hồ sơ công khai từ user.

## Participants

- web (client): Web client
- friend (service): Friend
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> friend [call/http]: GET /profiles/:profileUserId/follow-counts: lấy số follower/following (cả của chính mình) {apps/core-api/src/modules/friend/controllers/profile-social.controller.ts:46}
2. friend -> user [call/di-call]: Kiểm tra profile tồn tại (getByIdOrThrow) {apps/core-api/src/modules/friend/services/profile-social.service.ts:398}
3. friend -> safety [call/di-call]: Nếu xem profile của người khác: kiểm block 2 chiều, bị block thì 404 gộp {apps/core-api/src/modules/friend/services/profile-social.service.ts:400}
4. friend -> postgres [call/shared-db]: Đếm follow active theo hai chiều {apps/core-api/src/modules/friend/services/profile-social.service.ts:406}
5. friend -> web [return/http]: Trả followerCount và followingCount
6. web -> friend [call/http]: GET /profiles/:profileUserId/followers hoặc /following: lấy một trang danh sách {apps/core-api/src/modules/friend/controllers/profile-social.controller.ts:61, apps/core-api/src/modules/friend/controllers/profile-social.controller.ts:82}
7. friend -> safety [call/di-call]: Lấy tập user bị block 2 chiều với người xem để loại khỏi trang (Trước đó friend cũng gọi assertProfileReadable (getByIdOrThrow + assertNotBlocked) như ở follow-counts.) {apps/core-api/src/modules/friend/services/profile-social.service.ts:323}
8. friend -> postgres [call/shared-db]: Đọc profile_follows active theo keyset (last_followed_at, id), chỉ user active {apps/core-api/src/modules/friend/services/profile-social.service.ts:318}
9. friend -> user [call/di-call]: Lấy hồ sơ công khai của các user trong trang {apps/core-api/src/modules/friend/controllers/profile-social.controller.ts:167}
10. friend -> postgres [call/shared-db]: Đánh dấu người xem đang theo dõi ai trong trang (isFollowing) {apps/core-api/src/modules/friend/controllers/profile-social.controller.ts:173}
11. friend -> web [return/http]: Trả trang {profile, followedAt, isFollowing} kèm cursor kế tiếp (Cursor sai trả 400 FRIEND_CURSOR_INVALID.)
12. web -> friend [call/http]: GET /friends/connections: lấy danh sách bạn bè {apps/core-api/src/modules/friend/friend.controller.ts:85}
13. friend -> safety [call/di-call]: Lấy tập user bị block 2 chiều với mình {apps/core-api/src/modules/friend/services/profile-social.service.ts:214}
14. friend -> postgres [call/shared-db]: SQL gộp friendships và follow hai chiều, loại user bị khoá hoặc bị block (tối đa 500) {apps/core-api/src/modules/friend/services/profile-social.service.ts:229, apps/core-api/src/modules/friend/services/profile-social.service.ts:234}
15. friend -> user [call/di-call]: Lấy hồ sơ công khai của các đối phương {apps/core-api/src/modules/friend/friend.controller.ts:95}
16. friend -> web [return/http]: Trả danh sách bạn kèm cờ isFriend, isMutualFollow và since

## Invariants

- Cả bốn route đều lọc người có block 2 chiều với người xem: follow-counts/followers/following trả 404 khi chính profile bị block, danh sách và connections loại từng dòng bị block.
- Chỉ follow còn active và user active mới xuất hiện; unfollow là xoá mềm nên không làm lệch số đếm.
- Friend chỉ giữ id; hồ sơ công khai luôn do UserService ghép vào, nên danh sách không lộ trường ngoài PublicProfileDto.

## Failure modes

- when Profile không tồn tại hoặc bị block 2 chiều với người xem: follow-counts/followers/following trả 404 gộp (PROFILE_SOCIAL_PROFILE_NOT_AVAILABLE), không đọc bảng follow.
- when Cursor của followers/following sai định dạng: Trả 400 FRIEND_CURSOR_INVALID, không truy vấn trang.
- when Người dùng có hơn 500 bạn: friends/connections cắt ở 500 dòng mới nhất, không có phân trang.

## Uncertain

- Luồng này được thêm 2026-10-10 theo code HEAD d3cb28c bằng đọc mã trực tiếp; chưa qua bước kiểm toán độc lập của agent thứ hai như 71 flow trước.

## Endpoints covered

- GET /profiles/:profileUserId/follow-counts [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- GET /profiles/:profileUserId/followers [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- GET /profiles/:profileUserId/following [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- GET /friends/connections [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
