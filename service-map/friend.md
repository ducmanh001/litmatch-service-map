# Service map: Friend (+ Streak, Conversation, Profile-social)

id: service-map:friend
module: module:friend
deployable: core-api
confidence: high

Bạn bè, chat 1-1, streak, follow và mở chat từ profile

Sở hữu Friendship, Conversation/Message chat 1-1 lâu dài, trạng thái đọc/mute theo thành viên, ConversationStreak (sub-service nội bộ), ProfileFollow và ProfileChatContact (cổng first-contact của chat trực tiếp từ profile). KHÔNG sở hữu User/profile, block (Safety), ví/ledger (Economy), notification hay hạ tầng realtime; các flow match (Soul/Voice/Movie/Palm) chỉ tạo bạn qua FriendService.ensureFriendship.

Owns: bảng friendships, conversations, messages (append-only, cursor seq, idempotency_key unique); bảng conversation_member_states (last_read_at, muted_at), conversation_streaks; bảng profile_follows, profile_chat_contacts (first_contact_date UTC); job friend-streak-warning (advisory lock litmatch:cluster-job:friend-streak-warning)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/profiles/:profileUserId/actions | jwt | Trả isFollowing, follower/following count, conversationId hiện có và requiresGift (số ProfileChatContact hôm nay UTC >= ngưỡng); 400 khi tự xem mình, 404 gộp khi bị block 2 chiều. |
| e2 | rest | POST | /api/v1/profiles/:profileUserId/follow | jwt | Follow một chiều, upsert idempotent theo cặp (follower, followee) đặt active=true; chặn tự follow (400) và block 2 chiều (404). |
| e3 | rest | DELETE | /api/v1/profiles/:profileUserId/follow | jwt | Unfollow chỉ đặt active=false (giữ lịch sử last_followed_at); cùng guard tự thao tác và block như follow. |
| e4 | rest | POST | /api/v1/profiles/:profileUserId/conversation | jwt | Mở chat trực tiếp từ profile: transaction khoá hàng User đích, trả conversation sẵn có, nếu đã đủ N first-contact hôm nay trả 402 PROFILE_SOCIAL_MESSAGE_GIFT_REQUIRED, ngược lại tạo Conversation + ProfileChatContact. |
| e5 | rest | GET | /api/v1/friends | jwt | Inbox mọi conversation của caller sort theo lastMessageAt, kèm profile công khai, unreadCount, lastMessagePreview, muted, isFriend và canCall (follow active 2 chiều). |
| e6 | rest | GET | /api/v1/friends/:friendUserId/conversation | jwt | Tra conversation của một cặp theo cặp canonical; chưa có thì 404 NOT_FRIEND (không phân biệt user không tồn tại, tránh oracle). |
| e7 | rest | GET | /api/v1/conversations/:id/messages | jwt | Liệt kê message theo cursor keyset seq; chỉ 2 thành viên, conversation không tồn tại hoặc không phải thành viên đều 404 gộp. Là fallback REST khi socket rớt. |
| e8 | rest | GET | /api/v1/conversations/:id/streak | jwt | Streak hiện tại derive khi đọc (isActive khi gap <= 2 ngày UTC, chưa có streak trả current=0); chỉ thành viên. |
| e9 | rest | POST | /api/v1/conversations/:id/read | jwt | Đánh dấu đã đọc tới now() bằng upsert ON CONFLICT trên conversation_member_states; idempotent, chỉ thành viên. |
| e10 | rest | POST | /api/v1/conversations/:id/mute | jwt | Bật/tắt mute hội thoại (body {muted}); chỉ tắt kênh notification friend_message, message/realtime/unread vẫn chạy. |
| e11 | rest | POST | /api/v1/conversations/:id/messages | jwt | Gửi message: bắt buộc Idempotency-Key, guard thành viên + block 2 chiều (404 gộp), resolve imageAssetId qua Media, lưu message, tính streak trong transaction riêng, publish friend.message sau persist, notification best-effort (bỏ qua nếu người nhận mute). |
| e12 | socket-emit | EMIT | friend.message | internal | Báo tin mới cho cả người gửi lẫn đối phương qua realtime:user:{userId} với senderUserId/content/attachment (không ẩn danh); publish sau khi persist, best-effort, REST polling là fallback. |
| e13 | socket-emit | EMIT | friend.streak.at_risk | internal | Cảnh báo 'sắp mất streak' cho CẢ 2 thành viên (không biết ai chưa nhắn); do StreakWarningJob bắn, tối đa 1 lần/ngày UTC mỗi conversation nhờ lastWarningSentAt. |
| e14 | socket-emit | EMIT | friend.streak.increased | internal | Báo streak đạt mốc trong STREAK_MILESTONE_DAYS (currentStreak = milestoneHit) cho cả 2 thành viên; chỉ bắn khi trúng mốc, không phải mỗi lần tăng. |
| e15 | job | WORKER | StreakWarningJob.onApplicationBootstrap | internal | Job lặp mỗi STREAK_WARNING_CHECK_INTERVAL_MS, chỉ chạy trên 1 replica nhờ pg advisory lock; chọn conversation có streak > 0, lastConfirmedDate = hôm qua, qua giờ STREAK_WARNING_HOURS UTC; KHÔNG bao giờ ghi currentStreak/longestStreak. |
| e16 | rest | GET | /api/v1/friends/connections | jwt | Danh sách bạn bè = Friendship (cùng "Thích" lúc ghép đôi) hợp follow active hai chiều, gộp một dòng mỗi người kèm isFriend/isMutualFollow/since; không cần có conversation; loại user bị khoá và người có block 2 chiều; tối đa 500 dòng, không phân trang. |
| e17 | rest | GET | /api/v1/profiles/:profileUserId/follow-counts | jwt | Trả followerCount/followingCount của một profile, dùng được cho cả chính mình (khác /actions từ chối self); 404 gộp khi bị block 2 chiều. |
| e18 | rest | GET | /api/v1/profiles/:profileUserId/followers | jwt | Danh sách người theo dõi profile, cursor keyset (last_followed_at, id) mới theo dõi nhất trước, kèm isFollowing của người xem; chỉ follow active, user active, loại người có block 2 chiều với người xem; cursor sai trả 400. |
| e19 | rest | GET | /api/v1/profiles/:profileUserId/following | jwt | Danh sách profile mà user đang theo dõi, cùng cursor keyset và cùng bộ lọc block/active như followers. |

## Coordination edges (why this service talks to others)

### resolve ảnh chat (c1) with media [service, out, di-call]

- endpoints: e11
- what: FriendController gọi MediaService.resolveImageUrl(userId, imageAssetId, ImageAssetPurpose.Message) rồi đóng gói thành attachment {kind:'image', payload:{url}} trước khi gọi FriendService.sendMessage.
- why: DTO gửi tin chỉ nhận imageAssetId chứ không nhận URL; chỉ Media biết asset có thuộc đúng người gửi, đã upload và đúng MIME/size, nên Friend phải hỏi Media để không nhận URL tuỳ ý (docs/services/friend-service.md § 2).
- if missing: Người dùng nhét URL ảnh bất kỳ vào message, vượt kiểm tra chủ sở hữu/MIME, hoặc dùng assetId của người khác.
- evidence: apps/core-api/src/modules/friend/friend.controller.ts:210
- review: sound — Controller resolve imageAssetId qua Media rồi đóng gói attachment image; đúng như code. Lưu ý provider dev của Media bỏ qua bước kiểm MIME/size.

### thông báo tin nhắn/streak (c2) with notification [service, out, di-call]

- endpoints: e11, e15
- what: Sau khi gửi tin gọi NotificationService.create({type: FriendMessage, payload:{conversationId, senderUserId, preview}}) rồi sendPush cho người nhận; job cảnh báo tạo StreakAtRisk + sendPush cho từng thành viên.
- why: Người nhận đang offline không có socket; in-app notification và push là kênh duy nhất báo cho họ. Friend giữ quy tắc mute (đọc ConversationMemberState.mutedAt) nên tự quyết định có gọi Notification hay không; lỗi bị nuốt vì message đã commit.
- if missing: Tin nhắn và cảnh báo mất streak chỉ đến được người đang mở app; người nhận offline không biết có tin mới.
- evidence: apps/core-api/src/modules/friend/friend.service.ts:440, apps/core-api/src/modules/friend/friend.service.ts:449, apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:106
- review: sound — sendMessage và StreakWarningJob đều create + sendPush trong try/catch; mute chỉ áp cho friend_message, không áp cho cảnh báo streak.

### kiểm tra block 2 chiều (c3) with safety [service, out, di-call]

- endpoints: e1, e2, e3, e4, e11, e16, e17, e18, e19
- what: SafetyService.isBlocked(a,b) và isBlocked(b,a) trước sendMessage, trước follow/unfollow/getActions/openConversation (assertNotBlocked) và trong canCall. Danh sách mới: follow-counts/followers/following kiểm assertProfileReadable (404 khi block 2 chiều) và lọc getBlockedUserIds khỏi từng dòng; friends/connections cũng lọc getBlockedUserIds.
- why: Block là quy tắc của Safety, kiểm tại thời điểm hành động (docs/10 § 10.0.C). Cả hai chiều đều bị chặn và trả 404 không lộ ai block ai: sendMessage dùng FRIEND_CONVERSATION_NOT_FOUND như 'không phải thành viên'; thao tác profile dùng PROFILE_SOCIAL_PROFILE_NOT_AVAILABLE (khác PROFILE_NOT_FOUND của user không tồn tại).
- if missing: Người bị block vẫn nhắn, follow hoặc mở chat được với người đã chặn họ; mã lỗi khác biệt còn làm lộ danh sách block.
- evidence: apps/core-api/src/modules/friend/friend.service.ts:346, apps/core-api/src/modules/friend/friend.service.ts:347, apps/core-api/src/modules/friend/services/profile-social.service.ts:152, apps/core-api/src/modules/friend/services/profile-social.service.ts:214
- review: corrected — Chỉ nhánh chat trùng mã với 'không phải thành viên'; nhánh profile dùng mã riêng khác PROFILE_NOT_FOUND của user không tồn tại.

### kiểm tra/hydrate profile (c4) with user [service, out, di-call]

- endpoints: e1, e2, e3, e4, e5, e16, e17, e18, e19
- what: UserService.getByIdOrThrow(profileUserId) trong assertTarget của mọi thao tác profile; UserService.findByIds để dựng PublicProfileDto cho từng bạn ở GET /friends (getByIdOrThrow khi thiếu profile).
- why: Friend chỉ lưu userId; hồ sơ công khai và việc user có tồn tại là dữ liệu của User, nên phải hỏi User thay vì join bảng users (ranh giới module, AGENTS.md).
- if missing: Follow/mở chat tới userId không tồn tại tạo dữ liệu mồ côi; GET /friends không có tên/avatar để hiển thị.
- evidence: apps/core-api/src/modules/friend/friend.controller.ts:63, apps/core-api/src/modules/friend/friend.controller.ts:73, apps/core-api/src/modules/friend/services/profile-social.service.ts:398, apps/core-api/src/modules/friend/services/profile-social.service.ts:398
- review: sound — assertTarget gọi userService.getByIdOrThrow; listFriends dùng findByIds rồi getByIdOrThrow khi thiếu profile, đúng như mô tả.

### khoá hàng users đích (c5) with user [service, out, shared-db]

- endpoints: e4
- what: lockProfileForChatInManager: manager.findOne(User, {lock: pessimistic_write}) trên hàng của profile được nhắn, trước khi đếm ProfileChatContact hôm nay và insert Conversation/ProfileChatContact.
- why: Ngưỡng first-contact trong ngày là check-then-insert; hai viewer cùng bấm nhắn lúc count = N-1 cùng thấy 'còn chỗ' và vượt ngưỡng. Hàng users của profile là ổ khoá chung để serialize cả mở chat miễn phí (Friend) lẫn mở chat bằng quà (Gift/Economy).
- if missing: Race vượt ngưỡng: profile hot nhận nhiều hơn N first-contact miễn phí, cổng tặng quà bị bypass.
- evidence: apps/core-api/src/modules/friend/services/profile-social.service.ts:467
- review: sound — openConversation khoá hàng User (pessimistic_write) trước khi tìm conversation, đếm ProfileChatContact và insert; Gift gọi cùng hàm khoá trong withinTransaction.

### đẩy realtime tin/streak (c6) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e12, e13, e14
- what: publishRealtimeEvent(redis, logger, uid, envelope) cho friend.message (cả sender + partner), friend.streak.increased (khi trúng milestone) và friend.streak.at_risk (job), lên kênh realtime:user:{userId} qua RealtimeEnvelope.
- why: Core-api không giữ socket; signaling-gateway mới giữ kết nối đã xác thực JWT và fan-out xuống đúng user. Publish SAU khi DB commit và best-effort (nuốt lỗi) vì REST polling là fallback, không dùng outbox (docs/services/realtime-gateway.md).
- if missing: Tin nhắn chỉ xuất hiện khi client poll (5 giây), cảnh báo streak không tới được người đang mở app.
- evidence: apps/core-api/src/modules/friend/friend.service.ts:399, apps/core-api/src/modules/friend/friend.service.ts:425, apps/core-api/src/modules/friend/jobs/streak-warning.job.ts:100
- review: sound — publishRealtimeEvent nuốt lỗi và chạy sau persist; gateway psubscribe realtime:user:*; web poll 5_000ms. Đúng.

### client Redis publish riêng (c7) with redis [infra, out, redis-state]

- endpoints: e11, e15
- what: Provider FRIEND_REDIS tạo ioredis từ REDIS_URL (createCoreRedisClient); FriendService và StreakWarningJob dùng nó làm publisher; FriendModule đóng client ở onApplicationShutdown (closeCoreRedisClient).
- why: Publish pub/sub cần một kết nối Redis của chính module (docs/05 § 5.3) và phải đóng khi shutdown để không treo tiến trình. Module không lưu state nào trong Redis (state ở Postgres).
- if missing: Không có kết nối để PUBLISH nên không bắn được friend.message/friend.streak.*; kết nối không đóng làm shutdown treo.
- evidence: apps/core-api/src/modules/friend/friend.module.ts:69
- review: sound — friendRedisProvider tạo client từ REDIS_URL, module đóng ở onApplicationShutdown; Redis chỉ dùng để publish.

### bảng chat/streak/follow (c8) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e15, e16, e17, e18, e19
- what: Đọc/ghi friendships, conversations, messages, conversation_member_states, conversation_streaks, profile_follows, profile_chat_contacts; job lấy pg_try_advisory_xact_lock('litmatch:cluster-job:friend-streak-warning') để chỉ 1 replica chạy.
- why: Postgres là nguồn sự thật: unique(user_low,user_high) chặn tạo đôi, messages.idempotency_key unique làm replay an toàn, SELECT FOR UPDATE trên conversation_streaks serialize hai bên nhắn gần nhau để streak tăng đúng 1 lần/ngày; advisory lock tránh nhiều replica cùng cảnh báo.
- if missing: Có thể tạo đôi bạn/conversation, nhân đôi tin khi retry, tăng streak hai lần, hoặc cảnh báo streak lặp từ nhiều replica.
- evidence: apps/core-api/src/modules/friend/entities/conversation-member-state.entity.ts:12, apps/core-api/src/modules/friend/entities/conversation-streak.entity.ts:12, apps/core-api/src/modules/friend/entities/conversation.entity.ts:12
- review: sound — Đã xác nhận uq_friendships_pair, SELECT FOR UPDATE streak và uq_messages_idempotency_key (migration). Advisory lock chỉ loại chạy đồng thời; lặp giữa các tick chặn bằng lastWarningSentAt.

### quyền gọi + tạo bạn voice (c9) with calling [service, in, di-call]

- endpoints: e5
- what: CallingService.joinFriendCall gọi ProfileSocialService.canCall (block 2 chiều + follow active 2 chiều); voice match gọi FriendService.areFriends khi poll kết quả và ensureFriendship(manager, a, b, VoiceMatch) khi cả hai like.
- why: Quyền gọi lâu dài dựa trên follow/block do Friend sở hữu nên Calling phải hỏi Friend tại thời điểm mở phòng (canCall trong GET /friends chỉ là UX). Tạo bạn truyền manager của Calling để Friendship + Conversation nằm CÙNG transaction với reaction.
- if missing: Calling tự giữ bản sao follow/block có thể lệch; mutual-like voice có thể ghi reaction mà không có Friendship/Conversation.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:208, apps/core-api/src/modules/calling/calling.service.ts:297, apps/core-api/src/modules/calling/calling.service.ts:404
- review: sound — joinFriendCall gọi canCall; likeCall gọi ensureFriendship(manager) trong transaction khoá CallSession; areFriends khi poll kết quả. Khớp.

### quyền xem + reply story (c10) with feed [service, in, di-call]

- endpoints: e11, e6
- what: FeedService/StoryService gọi areFriends (audience Friends của post/story), listFriendIds (ring story), getConversationWithFriend + sendMessage với attachment kind 'story_reply' khi trả lời story.
- why: Audience 'friends' và ring story phụ thuộc đồ thị bạn do Friend sở hữu. Reply-to-story đi qua đúng pipeline idempotency, block, realtime và notification của chat; attachment do Feed đóng gói nên Friend không biết ngữ nghĩa story.
- if missing: Feed phải tự suy ra bạn bè (lệch nguồn sự thật) hoặc nhân bản logic gửi tin, bỏ qua block và streak.
- evidence: apps/core-api/src/modules/feed/feed.service.ts:159, apps/core-api/src/modules/feed/feed.service.ts:211, apps/core-api/src/modules/feed/services/story.service.ts:89
- review: sound — Đúng luồng. Lưu ý Conversation có thể tồn tại không cần Friendship (chat từ profile/quà) nên reply story không chỉ giới hạn ở bạn bè như comment trong code.

### cổng quà mở chat profile (c11) with gift [service, in, di-call]

- endpoints: e4, e1
- what: GiftService.sendProfileGift (trong transaction Economy) gọi assertNotBlocked, lockProfileForChatInManager, ensureConversationInManager và ensureProfileChatContactInManager với manager của giao dịch tặng quà.
- why: Quà là cách mở chat khi profile vượt ngưỡng first-contact (commit 6291078): Conversation + ProfileChatContact phải cùng transaction với GiftEvent/ledger để không trừ kim cương mà chat không mở. Gift phải khoá cùng hàng users với e4 để count N+1 không bị vượt.
- if missing: Trừ kim cương mà không mở chat (hoặc mở chat mà không ghi first-contact), hoặc quà và chat miễn phí chạy đua vượt ngưỡng.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:382, apps/core-api/src/modules/gift/gift.service.ts:417, apps/core-api/src/modules/gift/gift.service.ts:420
- review: sound — Thứ tự trong withinTransaction: assertNotBlocked, lock User, ghi GiftEvent, ensureConversation, ensureProfileChatContact, notification. Khoá chỉ bảo vệ đường đếm ngưỡng của chat miễn phí.

### kiểm tra bạn (mini-game) (c12) with mini-game [service, in, di-call]

- endpoints: 
- what: MiniGameService gọi FriendService.areFriends(userId, friendUserId) khi tạo MiniGameSession với bạn.
- why: Mini-game chỉ cho bạn bè; Friendship do Friend sở hữu nên mini-game hỏi Friend, không giữ bảng bạn riêng.
- if missing: Người lạ mời được người khác vào mini-game, không có chốt chặn quan hệ bạn.
- evidence: apps/core-api/src/modules/mini-game/mini-game.service.ts:71
- review: sound — MiniGameService.areFriends trả NOT_FRIEND 404 khi không phải bạn; đúng.

### kiểm tra + tạo bạn (movie) (c13) with movie-match [service, in, di-call]

- endpoints: e5, e6
- what: MovieMatchService gọi areFriends khi createSession xem chung với bạn và ensureFriendship(manager, low, high, MovieMatch) khi cả hai like trong phiên ẩn danh.
- why: Phiên xem chung theo bạn cần Friendship; mutual-like ẩn danh phải tạo bạn cùng transaction với việc chốt outcome để bất biến 'có Friendship thì có Conversation' đúng.
- if missing: Hai người like nhau nhưng Friendship không được tạo atomically, hoặc người lạ mời xem chung.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:123, apps/core-api/src/modules/movie-match/movie-match.service.ts:510
- review: sound — createSession kiểm areFriends; mutual-like anon truyền manager vào ensureFriendship cùng transaction chốt outcome.

### tạo bạn khi cùng like (palm) (c14) with palm-match [service, in, di-call]

- endpoints: e5, e6
- what: PalmMatchService gọi ensureFriendship(manager, low, high, PalmMatch) khi cả hai rating Like.
- why: Friend là chủ duy nhất của bảng friendships/conversations; Palm Match chỉ truyền manager để tạo cặp cùng transaction với outcome Matched.
- if missing: Palm Match phải ghi thẳng friendships (phá ownership) hoặc outcome Matched mà không có bạn.
- evidence: apps/core-api/src/modules/palm-match/palm-match.service.ts:286
- review: sound — PalmMatch truyền manager vào ensureFriendship khi cả hai Like; Friend ghi friendships/conversations bằng orIgnore.

### lọc video theo bạn bè (c15) with short-video [service, in, di-call]

- endpoints: 
- what: ShortVideoService.listPublished gọi FriendService.listFriendIds(userId) cho tab 'Đang theo dõi' rồi lọc authorUserId IN friendIds.
- why: Đồ thị bạn là nguồn 'follow' duy nhất của tab này (không có bảng follow riêng trong short-video), nên phải lấy danh sách từ Friend (giới hạn 500).
- if missing: Tab 'Đang theo dõi' không biết lọc theo ai.
- evidence: apps/core-api/src/modules/short-video/short-video.service.ts:206
- review: sound — listFriendIds giới hạn FRIEND_GRAPH_READ_LIMIT = 500 nên bạn thứ 501 trở đi không hiện ở tab Đang theo dõi.

### tạo bạn + import chat (c16) with soul-match [service, in, di-call]

- endpoints: e5, e6, e7
- what: SoulMatchService gọi areFriends (cờ matched, unlock partner) và ensureFriendship(manager, a, b, SoulMatch, messageSeeds) với FriendMessageSeed[] từ lịch sử chat ẩn danh khi cả hai like.
- why: Friendship là nguồn sự thật của unlock profile; Friendship + Conversation + message import phải cùng transaction của rating để mutual-like không mất và tin không bị bỏ giữa chừng. Friend nhận seed trung lập (key ổn định, ON CONFLICT DO NOTHING) nên replay không nhân đôi.
- if missing: Mutual-like mà không có Friendship/Conversation, mất lịch sử chat khi mở khoá, hoặc replay nhân đôi message.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:324, apps/core-api/src/modules/soul-match/soul-match.service.ts:340, apps/core-api/src/modules/soul-match/soul-match.service.ts:362
- review: sound — Seed có idempotencyKey ổn định friendHistoryImportKey, insert orIgnore trên unique idempotency_key; ensureFriendship nhận manager của rating.

### nghe friend.message + poll (c50) with web [client, both, socket]

- endpoints: e7, e12
- what: Web đăng ký useRealtimeEvent(FriendMessage) để invalidate query tin nhắn/inbox và hiện toast + Notification trình duyệt; đồng thời poll GET /conversations/:id/messages mỗi 5 giây (FRIEND_MESSAGES_REFETCH_INTERVAL_MS).
- why: Realtime là best-effort (publish sau commit, không outbox) nên client phải tự poll REST làm đường dự phòng thật khi socket rớt; không có poll thì tin bị publish lỗi sẽ không bao giờ hiện.
- if missing: Một lần publish Redis lỗi hoặc socket rớt làm tin nhắn không hiện cho tới khi người dùng tải lại.
- evidence: apps/web/src/features/friend-chat/api.ts:103, apps/web/src/features/friend-chat/components/conversation-thread.tsx:64
- review: sound — Toast và Notification trình duyệt nằm ở realtime-notification-bridge; thread invalidate query; poll 5s là fallback.

### Idempotency-Key khi gửi (c51) with web [client, in, http]

- endpoints: e11
- what: Composer sinh khóa bằng useIdempotencyKey và gửi header 'Idempotency-Key' cùng POST /conversations/:id/messages; backend lưu unique 'friend:msg:{userId}:{key}'.
- why: Mạng timeout làm client retry; khoá ổn định giữa các lần retry để unique idempotency_key (migration uq_messages_idempotency_key) trả lại message cũ thay vì chèn bản sao. Thiếu header thì 400. Lưu ý: nhánh replay vẫn chạy tiếp publish friend.message và notification nên có thể phát trùng.
- if missing: Retry sau timeout tạo tin nhắn trùng lặp và thêm realtime/notification thừa; streak không bị đếm đôi nhờ guard lastConfirmedDate theo ngày UTC.
- evidence: apps/web/src/features/friend-chat/api.ts:118, apps/web/src/features/friend-chat/components/message-composer.tsx:27
- review: corrected — Replay vẫn chạy tiếp publish friend.message và tạo notification; streak không bị đếm đôi nhờ guard lastConfirmedDate, kể cả khi thiếu key.

## Notes

- Doc vs code: friend-service.md § 2 nói Conversation.lastMessageAt cập nhật CÙNG transaction với insert message; code (ConversationService.sendMessage) save message rồi conversationRepo.update riêng, không bọc transaction (comment trong code ghi 'không cần cùng transaction' vì chỉ để sort).
- Doc vs code: streak-service.md § 7 nói trúng milestone thì publish friend.streak.increased + notification streak_milestone; code CHỈ publish realtime (FriendService.sendMessage), không có chỗ nào tạo NotificationType.StreakMilestone (enum có nhưng không có caller).
- Doc vs code: streak-service.md § 5 mô tả cron nằm trong FriendService; thực tế StreakWarningJob tự publish realtime + notification, StreakService chỉ trả danh sách conversationId và markWarningSent. Comment trong job còn trỏ nhầm '§ 4'.
- Nghi vấn: khi client retry cùng Idempotency-Key, ConversationService trả message cũ nhưng FriendService.sendMessage vẫn chạy tiếp (recordActivity, publish friend.message, tạo notification + push) nên replay có thể bắn trùng realtime/notification. Streak không tăng đôi nhờ guard lastConfirmedDate.
- friend-service.md nhắc POST /profiles/:profileUserId/gifts; route này KHÔNG nằm trong module friend mà ở gift (GiftService gọi ngược ProfileSocialService để khoá và ghi first-contact).
- Realtime của Friend publish bằng ioredis riêng (FRIEND_REDIS, REDIS_URL) truyền vào publishRealtimeEvent, không đi qua RedisRealtimePublisher DI của common/realtime.
- Mọi route dùng guard global JwtAuthGuard (APP_GUARD) vì không có @Public/@Roles nào trong module; không có throttle riêng cho gửi tin nhắn (khác Soul Match 60 req/phút).
- Web chưa có UI cho streak: không có handler friend.streak.* trong apps/web; chỉ có notification-copy cho streak_milestone/streak_at_risk.
