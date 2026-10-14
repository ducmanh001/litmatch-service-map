# Danh sách bạn, đọc tin và streak

id: flow:friend-list-and-conversation-reads
group: Xã hội & nội dung
services: module:friend, module:user

Màn chat cần danh sách hội thoại kèm hồ sơ đối phương, lịch sử tin nhắn, streak, mốc đã đọc và trạng thái tắt thông báo. Module friend giữ dữ liệu chat và gọi user để lấy hồ sơ công khai; các thao tác còn lại chỉ đọc/ghi bảng của friend sau khi kiểm tra thành viên.

Các lần đọc danh sách bạn, tin nhắn, streak và lần ghi mốc đã đọc/mute đều đi qua friend, trong đó chỉ danh sách bạn gọi sang user.

## Participants

- web (client): Web client
- friend (service): Friend
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> friend [call/http]: GET /friends: lấy danh sách hội thoại của tôi
2. friend -> postgres [call/shared-db]: Truy vấn Conversation của user (unread, mute, tin cuối) {apps/core-api/src/modules/friend/services/conversation.service.ts:68}
3. friend -> user [call/di-call]: Lấy hồ sơ công khai của các đối phương theo danh sách id {apps/core-api/src/modules/friend/friend.controller.ts:63}
4. friend -> user [call/di-call]: Nếu thiếu hồ sơ của một đối phương: gọi getByIdOrThrow (Controller chỉ gọi bước này khi có partnerId không nằm trong kết quả findByIds.) {apps/core-api/src/modules/friend/friend.controller.ts:73}
5. friend -> web [return/http]: Trả danh sách bạn kèm hồ sơ công khai
6. web -> friend [call/http]: GET /conversations/:id/messages: mở lịch sử chat
7. friend -> postgres [call/shared-db]: Kiểm tra thành viên Conversation rồi đọc Message theo cursor {apps/core-api/src/modules/friend/services/conversation.service.ts:59, apps/core-api/src/modules/friend/services/conversation.service.ts:215}
8. friend -> web [return/http]: Trả trang tin nhắn (404 nếu không phải thành viên)
9. web -> friend [call/http]: GET /conversations/:id/streak: xem streak hội thoại
10. friend -> postgres [call/shared-db]: Kiểm tra thành viên rồi đọc ConversationStreak {apps/core-api/src/modules/friend/services/conversation.service.ts:59, apps/core-api/src/modules/friend/services/streak.service.ts:113}
11. friend -> web [return/http]: Trả streak suy ra theo ngày UTC hiện tại, không ghi DB (getDisplayStreak so lastConfirmedDate với todayUtc(); gap quá 2 ngày thì hiển thị current=0 mà không sửa dòng DB.)
12. web -> friend [call/http]: POST /conversations/:id/read: đánh dấu đã đọc
13. friend -> postgres [call/shared-db]: Kiểm tra thành viên, upsert last_read_at, đọc lại trạng thái {apps/core-api/src/modules/friend/services/conversation.service.ts:59, apps/core-api/src/modules/friend/friend.service.ts:233}
14. friend -> web [return/http]: Trả lastReadAt và cờ muted
15. web -> friend [call/http]: POST /conversations/:id/mute: bật/tắt thông báo
16. friend -> postgres [call/shared-db]: Kiểm tra thành viên, upsert muted_at, đọc lại trạng thái {apps/core-api/src/modules/friend/services/conversation.service.ts:59, apps/core-api/src/modules/friend/friend.service.ts:250}
17. friend -> web [return/http]: Trả lastReadAt và cờ muted sau khi cập nhật

## Invariants

- Mọi thao tác theo conversationId đều gọi getConversationForMember trước khi đọc/ghi, nên người ngoài hội thoại nhận 404 gộp thay vì lộ sự tồn tại của conversation.
- Hồ sơ đối phương thuộc module user, nên friend chỉ giữ partnerId và phải gọi UserService để ghép PublicProfileDto vào danh sách bạn.
- Streak được suy ra khi đọc từ lastConfirmedDate và ngày UTC hiện tại; luồng đọc không ghi gì, việc reset thật chỉ xảy ra khi có tin nhắn mới.
- Mốc đã đọc và mute là upsert ON CONFLICT trên (conversation_id, user_id), nên gọi lặp hoặc hai tab đồng thời không tạo dòng trùng.

## Failure modes

- when Người gọi không phải thành viên hoặc conversation không tồn tại: getConversationForMember ném CONVERSATION_NOT_FOUND (HTTP 404), không đọc tin nhắn/streak và không ghi trạng thái.
- when Hồ sơ của một partnerId không có trong kết quả findByIds: Controller gọi getByIdOrThrow cho id thiếu đó, và nếu vẫn không có thì danh sách bạn không được dựng.

## Endpoints covered

- GET /friends [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- GET /conversations/:id/messages [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- GET /conversations/:id/streak [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- POST /conversations/:id/read [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- POST /conversations/:id/mute [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
