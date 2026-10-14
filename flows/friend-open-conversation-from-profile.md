# Mở chat từ hồ sơ trước khi là bạn

id: flow:friend-open-conversation-from-profile
group: Xã hội & nội dung
services: module:friend, module:user, module:safety

Người xem bấm nhắn tin trên hồ sơ của người lạ. friend phải hỏi user (hồ sơ có tồn tại) và safety (có block không), rồi khoá dòng user đích trong một transaction để việc đếm first-contact trong ngày và việc tạo Conversation không bị tách đôi khi nhiều người bấm cùng lúc.

Luồng POST /profiles/:profileUserId/conversation kiểm tra hồ sơ và block, khoá user đích rồi mới đếm ngưỡng và tạo Conversation, sau đó client đọc lại bằng GET /friends/:friendUserId/conversation.

## Participants

- web (client): Web client
- friend (service): friend
- user (service): user
- safety (service): safety
- postgres (infra): PostgreSQL

## Steps

1. web -> friend [call/http]: POST /profiles/:profileUserId/conversation (JWT) (Mọi route REST đều qua JwtAuthGuard toàn cục trừ route @Public.)
2. friend -> user [call/di-call]: Kiểm tra hồ sơ đích tồn tại (getByIdOrThrow) {apps/core-api/src/modules/friend/services/profile-social.service.ts:398}
3. friend -> safety [call/di-call]: Hỏi isBlocked theo cả hai chiều viewer/profile (Nếu bị block theo bất kỳ chiều nào, service ném PROFILE_NOT_AVAILABLE (404) trước khi mở transaction.) {apps/core-api/src/modules/friend/services/profile-social.service.ts:546, apps/core-api/src/modules/friend/services/profile-social.service.ts:547}
4. friend -> postgres [call/shared-db]: Khoá dòng User đích (pessimistic_write) {apps/core-api/src/modules/friend/services/profile-social.service.ts:467, apps/core-api/src/modules/friend/services/profile-social.service.ts:469}
5. friend -> postgres [call/shared-db]: Tìm Conversation theo cặp canonical; có sẵn thì trả luôn {apps/core-api/src/modules/friend/services/profile-social.service.ts:594}
6. friend -> postgres [call/shared-db]: Đếm ProfileChatContact first-contact trong ngày của profile (Nếu số đếm >= PROFILE_DIRECT_MESSAGE_DAILY_FIRST_CHAT_THRESHOLD thì ném MESSAGE_GIFT_REQUIRED (402) và không tạo gì.) {apps/core-api/src/modules/friend/services/profile-social.service.ts:608}
7. friend -> postgres [call/shared-db]: INSERT Conversation (orIgnore), đọc lại bản ghi {apps/core-api/src/modules/friend/services/profile-social.service.ts:497, apps/core-api/src/modules/friend/services/profile-social.service.ts:506}
8. friend -> postgres [call/shared-db]: INSERT ProfileChatContact (orIgnore) nếu vừa tạo mới {apps/core-api/src/modules/friend/services/profile-social.service.ts:467}
9. friend -> web [return/http]: Trả Conversation (ConversationDto) sau khi commit
10. web -> friend [call/http]: GET /friends/:friendUserId/conversation để đọc lại hội thoại
11. friend -> postgres [call/shared-db]: Tìm Conversation theo cặp canonical (findOneBy) (Không thấy Conversation thì ném FriendErrors.NOT_FRIEND (404).) {apps/core-api/src/modules/friend/services/conversation.service.ts:55}
12. friend -> web [return/http]: Trả Conversation của cặp

## Invariants

- Khoá pessimistic_write trên dòng User đích khiến mọi lần mở chat tới cùng một profile xếp hàng: đếm first-contact và insert Conversation/ProfileChatContact diễn ra trọn vẹn trước lần kế tiếp, nên ngưỡng không bị vượt do chạy song song.
- Kiểm tra tồn tại hồ sơ (user) và block (safety) chạy trước transaction vì là câu hỏi đồng bộ của module khác; chỉ phần ghi dữ liệu của friend nằm trong transaction để không giữ khoá khi gọi module khác.
- Conversation và ProfileChatContact được tạo cùng một transaction nên không có Conversation mới mà thiếu dấu first-contact để đếm ngưỡng.

## Failure modes

- when Viewer trùng profile hoặc profile không tồn tại: assertTarget ném SELF_PROFILE_ACTION (400) hoặc getByIdOrThrow ném lỗi; transaction chưa mở.
- when Hai bên có block theo bất kỳ chiều nào: Trả PROFILE_NOT_AVAILABLE (404), không lộ việc bị block.
- when Profile đã đạt ngưỡng first-contact trong ngày và chưa có Conversation: Ném MESSAGE_GIFT_REQUIRED (402) kèm dailyFirstChatCount và firstChatThreshold; transaction rollback, không tạo Conversation.
- when Đã có Conversation giữa hai người: Trả Conversation hiện có, không tính thêm first-contact.
- when GET /friends/:friendUserId/conversation khi chưa có Conversation: Ném NOT_FRIEND (404) 'Chưa có cuộc trò chuyện với profile này'.

## Endpoints covered

- POST /profiles/:profileUserId/conversation [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
- GET /friends/:friendUserId/conversation [module:friend] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/friend.md
