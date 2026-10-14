# Service map: Feed (+ Stories)

id: service-map:feed
module: module:feed
deployable: core-api
confidence: high

Bài đăng, bình luận, thả tim và Stories 24h

Sở hữu Post/Comment/Reaction (feed công khai toàn cục, không fanout) và Story/StoryView (ephemeral, hết hạn lọc lúc đọc). KHÔNG sở hữu quan hệ bạn bè/block/privacy/ảnh/notification: mọi quyết định xem được hay không đều hỏi module chủ qua DI; Feed không có inbound caller nào.

Owns: bảng posts (likeCount/commentCount denormalized, audience, deletedAt); bảng comments, reactions (unique postId+userId); bảng stories, story_views (unique storyId+viewerId)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/stories | jwt | Đăng story từ mediaAssetId: bắt buộc Idempotency-Key (unique DB), guest bị chặn, URL ảnh chỉ nhận qua Media.resolveImageUrl, hết hạn sau STORY_TTL_HOURS. |
| e2 | rest | GET | /api/v1/stories/ring | jwt | Ring story còn hạn của mình + bạn bè (listFriendIds), loại tác giả block 2 chiều và tác giả ẩn hồ sơ. |
| e3 | rest | GET | /api/v1/stories/:storyId | jwt | Xem 1 story và ghi StoryView (self-view không đếm, xem lại idempotent); block/hidden/sai audience đều trả 404 STORY_NOT_FOUND. |
| e4 | rest | GET | /api/v1/stories/:storyId/viewers | jwt | Danh sách người xem story, CHỈ tác giả truy vấn được; lọc block hiện tại lúc đọc. |
| e5 | rest | POST | /api/v1/stories/:storyId/reply | jwt | Trả lời story thành DM qua FriendService.sendMessage (attachment story_reply); cần đã có Conversation với tác giả (thường là bạn, hoặc mở trước qua profile chat/gift), nếu không 403 STORY_REPLY_REQUIRES_FRIENDSHIP. |
| e6 | rest | POST | /api/v1/feed/posts | jwt | Đăng bài: Idempotency-Key (unique DB, khác nội dung 409), guest bị chặn, ảnh chỉ qua imageAssetId được Media resolve, audience mặc định public. |
| e7 | rest | GET | /api/v1/feed/posts | jwt | Feed công khai toàn cục (chỉ audience=public), cursor seq giảm dần, loại tác giả đang block/bị block. |
| e8 | rest | GET | /api/v1/feed/users/:userId/posts | jwt | Timeline một tác giả: audience khớp quan hệ (chính mình thấy hết, bạn thấy public+friends, người lạ chỉ public); block/hidden trả như không tồn tại. |
| e9 | rest | GET | /api/v1/feed/posts/:postId | jwt | Chi tiết 1 bài qua getPostOrThrow: đã xoá, bị block, tác giả ẩn hồ sơ hay sai audience đều 404 POST_NOT_FOUND (chống oracle). |
| e10 | rest | DELETE | /api/v1/feed/posts/:postId | jwt | Xoá mềm bài (deletedAt), chỉ tác giả; đi qua getPostOrThrow nên cũng áp block/audience. |
| e11 | rest | POST | /api/v1/feed/posts/:postId/comments | jwt | Bình luận: guest bị chặn, guard block/audience, insert Comment + tăng commentCount + tạo Notification trong cùng transaction, sendPush sau commit. |
| e12 | rest | GET | /api/v1/feed/posts/:postId/comments | jwt | Liệt kê comment cũ -> mới theo cursor seq sau khi qua getPostOrThrow, kèm batch author PublicProfileDto. |
| e13 | rest | DELETE | /api/v1/feed/comments/:commentId | jwt | Xoá mềm comment, chỉ tác giả comment; chỉ request chuyển deletedAt NULL -> timestamp mới giảm commentCount (chống trừ đôi khi race). |
| e14 | rest | POST | /api/v1/feed/posts/:postId/reactions | jwt | Thả tim idempotent: insert Reaction + tăng likeCount + tạo Notification post_liked cùng transaction; thua race unique thì trả replay, không đếm đôi; guest bị chặn. |
| e15 | rest | DELETE | /api/v1/feed/posts/:postId/reactions | jwt | Bỏ thả tim idempotent: xoá Reaction và chỉ giảm likeCount khi thực sự xoá được dòng. |
| e16 | rest | GET | /api/v1/feed/posts/:postId/reactions | jwt | Trạng thái thả tim của chính mình + tổng likeCount, qua getPostOrThrow. |
| e17 | job | WORKER | StorySweeperService.onApplicationBootstrap | internal | Job StorySweeperService (theo STORY_SWEEPER_INTERVAL_MS) hard-delete story hết hạn theo lô 500, chạy 1 instance toàn cluster bằng clusterSingleton; chỉ dọn rác, correctness do filter expiresAt lúc đọc. |

## Coordination edges (why this service talks to others)

### quan hệ bạn bè / DM (c1) with friend [service, out, di-call]

- endpoints: e2, e3, e5, e8, e9, e10, e11, e12, e14, e15, e16
- what: FriendService.areFriends (audience friends), listFriendIds (ring story), getConversationWithFriend + sendMessage (reply story thành DM kèm attachment story_reply).
- why: Quan hệ bạn bè là dữ liệu do Friend sở hữu nên Feed không tự suy ra được ai xem được bài/story audience=friends. Reply story phải là DM thật: Conversation/Message/idempotency/block/realtime do Friend sở hữu, và mediaUrl phải snapshot vào message vì story chết sau TTL còn message sống mãi.
- if missing: Feed phải tự giữ bản sao danh sách bạn: bài friends rò cho người lạ hoặc ẩn với bạn; reply story phải tự tạo kênh chat riêng, bỏ qua block và realtime của Friend Chat.
- evidence: apps/core-api/src/modules/feed/feed.service.ts:159, apps/core-api/src/modules/feed/feed.service.ts:211, apps/core-api/src/modules/feed/services/story.service.ts:89
- review: sound — areFriends, listFriendIds và getConversationWithFriend + sendMessage(attachment story_reply, snapshot mediaUrl) đúng code. Conversation có thể có ngoài Friendship.

### resolve ảnh post/story (c2) with media [service, out, di-call]

- endpoints: e1, e6
- what: MediaService.resolveImageUrl(userId, assetId, ImageAssetPurpose.Post|Story) đổi imageAssetId/mediaAssetId thành URL final đã kiểm chứng trước khi lưu Post.imageUrl / Story.mediaUrl.
- why: DTO chỉ nhận assetId, không nhận URL tuỳ ý (feed-service.md § 6). Chỉ Media biết asset có thuộc đúng caller, đã upload xong và đúng MIME/size nên Feed bắt buộc hỏi Media rồi mới ghi URL vào bản ghi.
- if missing: Feed phải tin URL do client gửi: user gắn được ảnh của người khác, ảnh chưa kiểm hoặc URL ngoài hệ thống vào bài/story.
- evidence: apps/core-api/src/modules/feed/controllers/story.controller.ts:57, apps/core-api/src/modules/feed/feed.controller.ts:66
- review: sound — Controller resolveImageUrl với purpose Post/Story; guest không resolve mà service từ chối bằng GUEST_FORBIDDEN.

### thông báo like/comment (c3) with notification [service, out, di-call]

- endpoints: e11, e14
- what: NotificationService.createWithManager(manager, {type post_liked | post_commented, payload}) trong transaction của Feed, rồi sendPush(notification) sau commit (bỏ qua tự like/tự comment bài mình).
- why: Notification ghi cùng EntityManager nên like/comment và thông báo commit hoặc rollback cùng nhau (không có thông báo ma, không mất thông báo). Push là best-effort nên gọi SAU commit và sendPush không ném lỗi, để lỗi provider không làm hỏng like/comment đã lưu.
- if missing: Tác giả không biết có người tương tác với bài; hoặc nếu tạo ngoài transaction thì like rollback vẫn để lại thông báo, còn commit thành công có thể mất thông báo.
- evidence: apps/core-api/src/modules/feed/feed.service.ts:260, apps/core-api/src/modules/feed/feed.service.ts:275, apps/core-api/src/modules/feed/feed.service.ts:360
- review: sound — createWithManager trong transaction comment và like, bỏ qua tự tương tác, sendPush sau commit, sendPush không throw. Đúng.

### kiểm tra block 2 chiều (c4) with safety [service, out, di-call]

- endpoints: e2, e3, e4, e5, e7, e8, e9, e10, e11, e12, e14, e15, e16
- what: SafetyService.isBlocked(a,b) cho từng cặp (người xem, tác giả) và getBlockedUserIds(userId) để lọc hàng loạt feed/ring/viewers.
- why: Block là dữ liệu append-only do Safety sở hữu (docs/10 Trust & Safety). Feed phải cắt mọi điểm chạm giữa hai người block nhau và trả cùng 404 như không tồn tại để không lộ ai block ai; list dùng getBlockedUserIds một query thay vì gọi lặp cho từng tác giả.
- if missing: Người đã block vẫn thấy, comment, thả tim bài của nhau và xuất hiện trong ring/danh sách người xem story: vi phạm quy tắc Trust & Safety.
- evidence: apps/core-api/src/modules/feed/feed.service.ts:103, apps/core-api/src/modules/feed/feed.service.ts:150, apps/core-api/src/modules/feed/feed.service.ts:151
- review: sound — getPostOrThrow/getStoryOrThrow gộp 404; danh sách dùng getBlockedUserIds, timeline trả trang rỗng. Block là log append-only trong Safety.

### tác giả + privacy hồ sơ (c5) with user [service, out, di-call]

- endpoints: e2, e3, e5, e6, e7, e8, e9, e10, e11, e12, e14, e15, e16
- what: UserService.getByIdOrThrow/findByIds dựng author PublicProfileDto cho post/comment; PrivacySettingsService.isHidden/findForUsers (hideProfile) để ẩn bài/story của tác giả ẩn hồ sơ.
- why: Hồ sơ và cài đặt riêng tư do User sở hữu: Feed phải lấy PublicProfileDto (không tự chế DTO lộ birth date, trust score...) và batch-load một trang để web khỏi gọi GET /users/:id N lần. Tác giả bật hideProfile thì bài/story của họ phải biến mất với người khác.
- if missing: Client phải gọi N lần để lấy tác giả, hoặc Feed lộ field riêng tư; bài/story của người ẩn hồ sơ vẫn hiện ra.
- evidence: apps/core-api/src/modules/feed/feed.controller.ts:208, apps/core-api/src/modules/feed/feed.controller.ts:215, apps/core-api/src/modules/feed/feed.service.ts:153
- review: sound — PublicProfileDto chỉ gồm id, nickname, gender, avatarId, interests nên không lộ birth date/trust; batch findByIds; privacy isHidden/findForUsers đúng.

### posts/comments/stories (c6) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17
- what: Ghi/đọc posts, comments, reactions, stories, story_views; likeCount/commentCount tăng giảm trong cùng transaction với insert/delete Reaction/Comment; job sweeper hard-delete stories và story_views cascade theo FK.
- why: Unique (postId,userId) và unique idempotencyKey trong DB là thứ duy nhất chặn double-like và đăng trùng khi retry/race. Counter denormalized chỉ đúng nếu cập nhật nguyên tử cùng dòng Reaction/Comment (docs/10 § Feed), nên bắt buộc dùng chung một transaction.
- if missing: Like đôi, bài đăng trùng khi mất mạng retry, likeCount/commentCount lệch số dòng thật.
- evidence: apps/core-api/src/modules/feed/entities/comment.entity.ts:10, apps/core-api/src/modules/feed/entities/post.entity.ts:26, apps/core-api/src/modules/feed/entities/reaction.entity.ts:14
- review: sound — uq_reactions_post_user, uq_posts_idempotency_key (partial WHERE NOT NULL), uq_stories_idempotency_key và story_views ON DELETE CASCADE đều có; counter cùng transaction.

### upload ảnh + Idempotency-Key (c50) with web [client, in, http]

- endpoints: e1, e6, e7
- what: Web upload ảnh qua Media (uploadImage(file,'post') lấy assetId) rồi POST /feed/posts kèm header Idempotency-Key sinh theo intent; sau khi tạo post thì invalidate query feed để tải lại danh sách.
- why: Feed không nhận binary hay URL: client phải upload trước để có assetId hợp lệ. Server không đẩy realtime cho Feed nên web chỉ thấy bài/comment mới bằng cách refetch REST; Idempotency-Key giúp retry khi mạng lỗi không đăng đôi bài.
- if missing: Client không có assetId để gắn ảnh, retry mạng đăng trùng bài, và feed không tự cập nhật sau khi đăng.
- evidence: apps/web/src/features/feed/api.ts:71, apps/web/src/features/feed/components/post-composer.tsx:110
- review: sound — Feed không publish realtime; web gửi Idempotency-Key, upload qua Media lấy assetId rồi invalidate feedKeys.list.

## Notes

- Doc feed-service.md § 3/§ 8 chỉ nhắc block và audience; code còn chặn thêm theo PrivacySettings.hideProfile (privacySettings.isHidden) ở getPostOrThrow, listUserTimeline, getStoryOrThrow và getRing: tác giả ẩn hồ sơ thì người khác nhận 404 như không tồn tại. Doc chưa mô tả cạnh Feed -> User này.
- Doc § 4 nói guest bị chặn bằng AuthenticatedUser.isGuest; code đúng vậy cho post/comment/like/story, nhưng ở controller guest còn bị bỏ qua bước resolveImageUrl (imageUrl=undefined) rồi service mới ném GUEST_FORBIDDEN.
- Tất cả route Feed/Story là jwt qua APP_GUARD JwtAuthGuard toàn cục (không @Public, không roles); không có endpoint admin nào gọi vào Feed (module exports: [] ).
- Story: StoryAudience chỉ có public|friends (không có only_me như Post). Ring chỉ gồm mình + bạn bè dù schema có audience=public (doc § 8 đã nói rõ).
- Feed không phát realtime event nào; thông báo like/comment đi qua Notification (in-app + push best-effort), client phải poll REST để thấy bài/comment mới.
