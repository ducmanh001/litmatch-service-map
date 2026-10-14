# Service map: Gift

id: service-map:gift
module: module:gift
deployable: core-api
confidence: high

Catalog quà và luồng tặng quà

Sở hữu catalog quà (bảng gifts, giá là DATA) và GiftEvent (gift_events) cho 3 ngữ cảnh tặng: phòng Party, tác giả video, mở chat từ profile; tính tỉ lệ PTS và điều phối side effect quanh giao dịch. KHÔNG sở hữu sổ cái/ví (Economy ghi LedgerEntry), không sở hữu Conversation/membership phòng/video/notification (mỗi module đó tự ghi).

Owns: bảng gifts (catalog: code, priceDiamond, sortOrder, active); bảng gift_events (1 event ↔ 1 transaction_id, đúng 1 context room_id|video_id|profile_user_id); ioredis client riêng GIFT_REDIS chỉ để publish gift.sent

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/gifts | jwt | Liệt kê catalog quà đang bật (active) với giá server là nguồn sự thật; client map code sang asset/animation. Cần JWT (JwtAuthGuard toàn cục, không @Public). |
| e2 | rest | POST | /api/v1/party/rooms/:roomId/gifts | jwt | Tặng quà trong phòng Party: bắt buộc Idempotency-Key, throttle 30/phút, cả người tặng và người nhận phải là member của phòng active; Economy trừ DIA/cộng PTS + ghi GiftEvent + notification trong 1 transaction, rồi publish gift.sent sau commit. |
| e3 | rest | POST | /api/v1/videos/:videoId/gifts | jwt | Tặng quà cho tác giả video: người nhận suy từ video.authorUserId (client không gửi receiverUserId), video phải nhìn thấy được; cùng chốt tiền/idempotency với quà phòng, không fanout realtime, người nhận biết qua notification gift_received. |
| e4 | rest | POST | /api/v1/profiles/:profileUserId/gifts | jwt | Tặng quà để mở chat trực tiếp từ profile: kiểm tra block hai chiều (trước và lại TRONG transaction), khoá user đích, rồi ledger + GiftEvent + Conversation + ProfileChatContact + notification cùng 1 transaction; retry idempotent không tạo unlock thứ hai. |
| e5 | socket-emit | EMIT | gift.sent | internal | Báo cho mọi member active của phòng có quà mới (giftCode, sender, receiver, priceDiamond, pointsAwarded) để client bắn hiệu ứng; chỉ publish khi KHÔNG phải replay, SAU commit, best-effort (phòng đóng giữa chừng thì chỉ log warn). |

## Coordination edges (why this service talks to others)

### trừ DIA, cộng PTS atomic (c1) with economy [service, out, di-call]

- endpoints: e2, e3, e4
- what: EconomyService.sendGift({senderUserId, receiverUserId, priceDiamond, pointsAwarded, idempotencyKey: gift:send:{sender}:{key}, metadata, withinTransaction}) ghi ledger: chân DIA Nợ user_wallet/Có system_gift_pool luôn có; chân PTS Nợ system_points_mint/Có user_earnings chỉ thêm khi pointsAwarded > 0. Callback withinTransaction(manager, transactionId) để Gift ghi GiftEvent + Notification (+ Conversation ở profile).
- why: Economy là nơi DUY NHẤT được ghi LedgerEntry/Wallet; 'trừ mà không cộng' hoặc có GiftEvent mà không có tiền là vi phạm bất biến tiền. Truyền manager qua callback để ledger, GiftEvent và side effect chung MỘT DB transaction; idempotencyKey unique ở Transaction làm retry trả lại GiftEvent cũ qua transaction_id.
- if missing: Gift phải tự ghi ví: có thể trừ DIA nhưng không ghi GiftEvent (hoặc ngược lại), double-charge khi client retry, và PTS có thể vượt giá quà (nhân đôi giá trị).
- evidence: apps/core-api/src/modules/gift/gift.service.ts:210, apps/core-api/src/modules/gift/gift.service.ts:308, apps/core-api/src/modules/gift/gift.service.ts:401
- review: corrected — Chân PTS chỉ được ghi khi pointsAwarded > 0 (economy.service:~625); guest hoặc làm tròn 0 thì chỉ còn chân DIA.

### chặn block + mở chat (c2) with friend [service, out, di-call]

- endpoints: e4
- what: ProfileSocialService.assertNotBlocked(sender, profile) (gọi cả trước và trong transaction), lockProfileForChatInManager(manager, profileUserId) khoá hàng users, ensureConversationInManager rồi ensureProfileChatContactInManager (chỉ khi conversation mới tạo) ghi Conversation + ProfileChatContact bằng cùng manager.
- why: Quà là đường thay thế khi chat miễn phí chạm ngưỡng first-chat/ngày: Gift không tự kiểm ngưỡng (MESSAGE_GIFT_REQUIRED chỉ ném ở openConversation, profile-social.service.ts:186-197). Gift dùng CÙNG lockProfileForChatInManager (khoá hàng users của profile) và cùng bảng ProfileChatContact (ensureProfileChatContactInManager, insert ON CONFLICT DO NOTHING) nên mọi first-contact vào một hồ sơ được tuần tự hoá và lượt mở bằng quà cũng được tính vào countDailyFirstChats của đường miễn phí. assertNotBlocked lặp lại trong transaction để quà không thành đường bypass Safety nếu block xảy ra giữa chừng.
- if missing: Có thể trả tiền tặng quà cho người đã block mình; ledger đã trừ mà Conversation không tạo (không atomic); hai first-contact song song cùng thắng hoặc đếm first-contact lệch so với ngưỡng của đường miễn phí.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:382, apps/core-api/src/modules/gift/gift.service.ts:417, apps/core-api/src/modules/gift/gift.service.ts:420
- review: sound — lockProfileForChatInManager khoá users pessimistic_write, openConversation dùng cùng khoá; ProfileChatContact orIgnore và countDailyFirstChats đếm theo bảng đó; MESSAGE_GIFT_REQUIRED chỉ ở openConversation.

### gift_received in-app + push (c3) with notification [service, out, di-call]

- endpoints: e2, e3, e4
- what: Trong withinTransaction: NotificationService.createWithManager(manager, {userId: receiver, type: GiftReceived, payload{roomId|videoId|profileUserId, senderUserId, giftCode, priceDiamond}}); sau commit (không replay) gọi NotificationService.sendPush(notification).
- why: Quà video/profile không có fanout realtime nên notification là cách DUY NHẤT người nhận biết; ghi bằng manager của giao dịch để notification tồn tại khi và chỉ khi tiền đã trừ. Push gọi sau commit vì best-effort, không được làm hỏng giao dịch tiền.
- if missing: Người nhận video/profile không bao giờ biết mình được tặng quà (hoặc nhận thông báo cho giao dịch đã rollback); push lỗi có thể làm rollback giao dịch tiền nếu gọi trong tx.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:238, apps/core-api/src/modules/gift/gift.service.ts:261, apps/core-api/src/modules/gift/gift.service.ts:335
- review: sound — Notification tạo bằng manager trong callback, sendPush gọi sau sendGift khi !replayed và sendPush nuốt mọi lỗi.

### kiểm tra member + fanout (c4) with party-room [service, out, di-call]

- endpoints: e2, e5
- what: PartyRoomService.getActiveRoomMembers(roomId): lúc gửi để xác nhận sender và receiver đều là member của phòng Active (ném ROOM_CLOSED nếu phòng đóng); lúc publishGiftSent lấy lại danh sách để fanout gift.sent cho từng member.
- why: Membership và trạng thái phòng thuộc Party Room, Gift không có bảng riêng nên phải hỏi owner. Danh sách fanout được đọc lại SAU commit để gửi đúng những người đang ở phòng lúc đó, không phải lúc bắt đầu request.
- if missing: Có thể tặng quà vào phòng đã đóng hoặc cho người không ở phòng; không biết gửi gift.sent cho ai nên hiệu ứng không tới khán giả.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:184, apps/core-api/src/modules/gift/gift.service.ts:495
- review: sound — Kiểm member là check nghiệp vụ (comment gift.service:181-182), publishGiftSent đọc lại getActiveRoomMembers sau commit và bọc try/catch.

### suy người nhận từ video (c5) with short-video [service, out, di-call]

- endpoints: e3
- what: ShortVideoService.getVideoOrThrow(user, videoId) trả Video; Gift lấy video.authorUserId làm receiverUserId. Lỗi VIDEO_NOT_FOUND nếu video không Published (trừ tác giả).
- why: Người nhận phải suy từ dữ liệu video (không tin client gửi receiverUserId) để không tặng lệch người, và quy tắc 'video nào nhìn thấy được' là của Short Video (guard trung tâm) nên Gift tái dùng thay vì tự đọc bảng videos.
- if missing: Client có thể gửi quà cho người bất kỳ dưới danh nghĩa 1 video, hoặc tặng quà cho video bị ẩn/chưa publish.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:280
- review: sound — getVideoOrThrow trả VIDEO_NOT_FOUND nếu không Published và không phải tác giả; receiver lấy từ video.authorUserId, tự tặng bị chặn.

### kiểm tra người nhận guest (c6) with user [service, out, di-call]

- endpoints: e2, e3, e4
- what: UserService.getByIdOrThrow(receiverUserId) để đọc receiver.isGuest; pointsAwarded = isGuest ? 0 : floor(priceDiamond * GIFT_POINTS_RATE_PERCENT / 100).
- why: docs/06 § Gift: guest không nhận điểm quy đổi (chân PTS = 0) nhưng quà vẫn tặng được; trạng thái guest nằm ở User nên Gift phải hỏi trước khi gọi Economy để truyền đúng pointsAwarded. Cũng làm 404 sớm nếu người nhận không tồn tại.
- if missing: Guest nhận PTS (vi phạm domain rule) hoặc tặng cho userId không tồn tại mới lỗi muộn ở tầng ledger.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:201, apps/core-api/src/modules/gift/gift.service.ts:299, apps/core-api/src/modules/gift/gift.service.ts:392
- review: sound — pointsAwarded = isGuest ? 0 : floor(price*rate/100) đúng ở các điểm gọi; getByIdOrThrow cho 404 sớm.

### PUBLISH gift.sent (c7) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e5
- what: publishRealtimeEvent(this.redis, logger, member.userId, envelope) PUBLISH envelope gift.sent lên Redis channel realtime:user:{userId} cho từng member active; signaling-gateway psubscribe và relay xuống socket.
- why: Gateway không có DB hay business logic nên core-api phải tính sẵn payload và chỉ publish SAU commit để không phát hiệu ứng cho giao dịch rollback (client chỉ bắn hiệu ứng sau khi server xác nhận). Best-effort vì REST vẫn là nguồn sự thật. Gift dùng ioredis client riêng GIFT_REDIS (đóng ở onApplicationShutdown).
- if missing: Khán giả khác trong phòng không thấy hiệu ứng quà cho đến khi tự reload dữ liệu; người tặng vẫn thấy qua response 200.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:498
- review: sound — realtimeUserChannel(userId) = realtime:user:{userId}, gateway psubscribe REALTIME_USER_CHANNEL_PATTERN, GIFT_REDIS đóng ở onApplicationShutdown trong gift.module.

### gifts / gift_events (c9) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4
- what: Đọc catalog gifts (findOneBy {id, active:true}, find active) và ghi gift_events cùng transaction ledger; replay đọc lại bằng findOneByOrFail({transactionId}).
- why: Giá quà là DATA trong DB, server đọc lại ĐÚNG lúc tặng (client chỉ gửi giftId); gift_events.transaction_id unique + CHECK chk_gift_events_context (đúng 1 ngữ cảnh) cho phép idempotent replay trả đúng event cũ.
- if missing: Giá do client quyết định (gian lận giá) và không có cách khớp một lần retry với event đã tạo, dẫn tới trừ tiền lặp.
- evidence: apps/core-api/src/modules/gift/entities/gift-event.entity.ts:14, apps/core-api/src/modules/gift/entities/gift.entity.ts:14
- review: sound — gift_events.transactionId unique và CHECK chk_gift_events_context có trong migration 1757500000000; giá đọc lại từ gifts tại lúc tặng.

### admin quản lý catalog (c10) with admin [service, in, di-call]

- endpoints: e1
- what: AdminService gọi GiftService.listAllForAdmin(), createGift(manager, input), updateGift(manager, id, input) trong dataSource.transaction cùng AuditLogService.record (gift.created/gift.updated); route admin cần quyền ManageGifts.
- why: Catalog nằm ở Gift nên admin chỉ được đổi qua public API của module; nhận manager để thay đổi quà và audit log commit cùng nhau, mọi thay đổi giá/active đều có vết. Giá đổi ở đây là giá mà e1/e2/e3/e4 đọc ở lần tặng kế tiếp.
- if missing: Không có cách vận hành đổi giá/tắt quà an toàn (phải sửa DB tay, không audit); hoặc audit log và thay đổi quà lệch nhau khi một bên rollback.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:271, apps/core-api/src/modules/admin/admin.service.ts:276, apps/core-api/src/modules/admin/admin.service.ts:297
- review: sound — createGift/updateGift nhận manager cùng audit; updateGift không xoá cứng, chỉ active=false; giá đọc lại ở sendGift sau đó.

### Idempotency-Key + hiệu ứng (c50) with web [client, both, http]

- endpoints: e1, e2, e3, e4, e5
- what: Web gửi POST /party/rooms/:roomId/gifts kèm header Idempotency-Key sinh theo intent (giữ nguyên qua retry) và lắng nghe RealtimeEvents.GiftSent để hiện hiệu ứng khi data.roomId khớp phòng đang xem.
- why: Idempotency-Key do client sinh là thứ duy nhất cho Economy phân biệt retry với lần tặng mới; còn hiệu ứng chỉ được bắn sau khi server xác nhận (200 hoặc gift.sent) để không hiển thị quà của giao dịch thất bại.
- if missing: Mỗi lần retry mạng là một lần trừ DIA mới (hoặc server từ chối vì thiếu header); client hiện hiệu ứng cho quà chưa thanh toán.
- evidence: apps/web/src/features/party-room/api.ts:304, apps/web/src/features/party-room/components/party-stage.tsx:177
- review: sound — Web gọi POST /party/rooms/{roomId}/gifts với Idempotency-Key và nghe RealtimeEvents.GiftSent khớp roomId.

## Notes

- Doc gift-service.md § 3 nói người tặng và người nhận phải là member active của phòng; code đúng vậy nhưng comment ở gift.service.ts:181 nói rõ đây chỉ là check nghiệp vụ, KHÔNG phải chốt an toàn tiền (tiền đúng nhờ transaction + idempotency ở Economy).
- Tặng quà cho video và profile KHÔNG publish gift.sent (chỉ quà phòng có fanout realtime); người nhận chỉ biết qua notification gift_received (in-app + push best-effort). Doc xác nhận cho video, riêng profile doc không nói nhưng code cũng không publish.
- ShortVideoService.getVideoOrThrow cho phép tác giả xem video chưa published của chính mình, sau đó sendVideoGift mới chặn SELF_GIFT_FORBIDDEN; người lạ chỉ tặng được video Published.
- Docs/ledger: economy.sendGift ghi thêm OutboxEvent 'economy.gift.sent' cùng transaction (do Economy sở hữu, Gift không chạm); gift.module.ts không import Kafka/outbox trực tiếp nên không có cạnh kafka ở đây.
- ProfileSocialService trong GiftService là @Optional-style (gift.service.ts:378 ném Error 'chưa được cấu hình' nếu thiếu); trong module thực FriendModule luôn được import nên nhánh lỗi chỉ phục vụ test/wiring.
- Replay cùng Idempotency-Key: ledger.recordInManager trả sớm (ledger.service.ts:106-114) và không chạy lại withinTransaction, nên Gift không tạo GiftEvent/Notification mới; GiftService đọc lại GiftEvent theo transaction_id (gift.service.ts:254-258) và chỉ publish gift.sent / sendPush khi !replayed.
- Gift không kiểm ngưỡng first-chat/ngày: MESSAGE_GIFT_REQUIRED chỉ ném ở ProfileSocialService.openConversation (profile-social.service.ts:186-197). Quà mở chat vẫn insert ProfileChatContact, nên mỗi lượt mở bằng quà cũng được tính vào countDailyFirstChats của đường miễn phí.
