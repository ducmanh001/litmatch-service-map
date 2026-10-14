# Service map: User

id: service-map:user
module: module:user
deployable: core-api
confidence: high

Hồ sơ người dùng, quyền riêng tư, trạng thái

Sở hữu bảng users (hồ sơ, status, role, isGuest, trustScore) và user_privacy_settings; cung cấp UserService/PrivacySettingsService/PublicProfileDto cho gần như mọi module khác. Không sở hữu danh tính đăng nhập (auth), không biết khái niệm block/report (safety) và không tự ghi presence (signaling-gateway ghi, user chỉ đọc).

Owns: bảng users (nickname, gender, birthDate, region, interests, seeking*, trustScore, status, isGuest, role); bảng user_privacy_settings (searchableByPhone, hideProfile, showOnlineStatus); đọc Redis ZSET presence realtimePresenceKey(userId) (do signaling-gateway ghi)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /api/v1/users/me | jwt | Đọc hồ sơ đầy đủ của chính mình (MyProfileDto) qua UserService.getByIdOrThrow. |
| e2 | rest | PATCH | /api/v1/users/me | jwt | Sửa hồ sơ của mình (nickname, gender, birthDate không ở tương lai, region, interests, seeking*); kiểm khoảng tuổi seeking sau khi gộp giá trị cũ và mới. |
| e3 | rest | GET | /api/v1/users/me/privacy | jwt | Đọc privacy settings của mình; chưa có row thì trả DEFAULT_PRIVACY_SETTINGS. |
| e4 | rest | PUT | /api/v1/users/me/privacy | jwt | Ghi privacy settings của mình bằng upsert theo userId (searchableByPhone, hideProfile, showOnlineStatus), giá trị thiếu quay về mặc định. |
| e5 | rest | GET | /api/v1/users/:id/presence | jwt | Cho biết user khác có đang online không: chỉ true khi họ bật showOnlineStatus VÀ Redis presence còn lease; không kiểm user tồn tại. |
| e6 | rest | GET | /api/v1/users/:id | jwt | Hồ sơ công khai tối thiểu (PublicProfileDto) của user khác theo id; không kiểm hideProfile/ban (xem notes). |

## Coordination edges (why this service talks to others)

### đọc presence online (c1) with redis [infra, out, redis-state]

- endpoints: e5
- what: RedisPresenceReader.isOnline: ZREMRANGEBYSCORE key -inf now để xoá lease hết hạn rồi ZCARD > 0 trên key realtimePresenceKey(userId); lỗi Redis trả false.
- why: Online/offline là trạng thái dẫn xuất từ socket lease của signaling-gateway, không có trong Postgres, nên user module phải đọc Redis để trả lời e5. Fail-closed vì báo online sai là lộ thông tin riêng tư, còn thiếu thì chỉ mất dấu chấm xanh.
- if missing: Không có chỉ báo online; nếu đọc fail-open (trả true khi Redis lỗi) thì Redis chập chờn làm user bật showOnlineStatus bị báo 'online' sai dù đã offline (người tắt hiển thị đã bị chặn trước đó ở controller).
- evidence: apps/core-api/src/modules/user/redis/redis-presence-reader.adapter.ts:23
- review: corrected — e5 đã short-circuit bằng settings.showOnlineStatus trước khi đọc Redis; fail-open chỉ gây báo online sai, không làm lộ người đã tắt hiển thị.

### users, privacy settings (c2) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6
- what: Đọc/ghi bảng users (findOneBy, save, find In, countBy, query builder cho browse/stats) và user_privacy_settings (upsert theo userId).
- why: Hồ sơ và privacy là dữ liệu bền vững của chính module; upsert theo khoá userId đảm bảo mỗi user đúng một dòng privacy, còn browsePage dùng NOT EXISTS trên hide_profile để lọc ngay trong SQL thay vì lọc sau phân trang.
- if missing: Không có nguồn sự thật về user; lọc hideProfile sau khi phân trang sẽ làm trang trả thiếu hoặc rỗng.
- evidence: apps/core-api/src/modules/user/entities/privacy-setting.entity.ts:4, apps/core-api/src/modules/user/entities/user.entity.ts:27
- review: sound

### admin: ban, role, list user (c3) with admin [service, in, di-call]

- endpoints: 
- what: AdminService gọi UserService.banUser/unbanUser(manager, id), findPage, getByIdOrThrow, listStaff, lockRoleAssignmentContext + setRoleWithManager, getAdminStats, listActiveUserIds, findByIds.
- why: Ban/unban nhận EntityManager của admin để đổi users.status và ghi audit log cùng một transaction (hành động nhạy cảm không được thành công một nửa). Đổi role khoá toàn bộ admin hiện có bằng pessimistic_write để hai request hạ quyền song song không cùng thấy 'còn admin khác' rồi hạ cả hai.
- if missing: Ban đổi status mà audit lỗi (hoặc ngược lại); race demote làm hệ thống không còn admin nào.
- evidence: apps/core-api/src/modules/admin/admin.service.ts:53, apps/core-api/src/modules/admin/admin.service.ts:61, apps/core-api/src/modules/admin/admin.service.ts:77
- review: sound

### auth: nạp/tạo user (c4) with auth [service, in, di-call]

- endpoints: 
- what: AuthService gọi getByIdOrThrow (kiểm status/isGuest/role trước khi cấp token, kể cả lúc refresh), createWithManager(manager, {nickname, isGuest}) và PrivacySettingsService.isSearchableByPhone cho search-by-phone.
- why: createWithManager nhận manager của auth để users và auth_identities được tạo cùng một transaction, tránh user mồ côi khi một bước fail. Khi refresh, auth hỏi lại status để user bị ban giữa hai lần refresh không nhận phiên mới.
- if missing: Có user không có identity (không đăng nhập lại được) hoặc ban không chặn được refresh.
- evidence: apps/core-api/src/modules/auth/auth.service.ts:128, apps/core-api/src/modules/auth/auth.service.ts:162, apps/core-api/src/modules/auth/auth.service.ts:170
- review: sound

### calling: region chọn LiveKit (c5) with calling [service, in, di-call]

- endpoints: 
- what: CallingService.resolveCallLivekitUrl gọi getByIdOrThrow cho userA và userB để lấy region khi LIVEKIT_REGION_URLS được bật.
- why: Region của hồ sơ quyết định cụm LiveKit nào phục vụ cuộc gọi; dữ liệu này nằm ở users nên calling phải hỏi user và dùng region userA khi hai bên lệch (quy tắc tất định).
- if missing: Không chọn được cụm LiveKit gần người dùng; mọi cuộc gọi rơi về LIVEKIT_URL mặc định.
- evidence: apps/core-api/src/modules/calling/calling.service.ts:583, apps/core-api/src/modules/calling/calling.service.ts:584
- review: sound

### discovery: duyệt user hợp lệ (c6) with discovery [service, in, di-call]

- endpoints: 
- what: DiscoveryService.browsePage(filter, limit, after) và NearbyService gọi findActiveByIds cùng PrivacySettingsService.findForUsers để lấy user active theo gender/tuổi/guest kèm privacy.
- why: Luật 'user nào được hiển thị' (active, không hideProfile, tuổi, guest) phải định nghĩa một chỗ duy nhất ở user; Discovery chỉ tự lọc theo vị trí/settings của nó và truyền excludeUserIds (self + block từ safety) vì user module không biết block/report.
- if missing: Hai nơi định nghĩa 'hợp lệ để hiện' lệch nhau: user banned hoặc hideProfile vẫn xuất hiện trong khám phá.
- evidence: apps/core-api/src/modules/discovery/discovery.service.ts:63, apps/core-api/src/modules/discovery/nearby.service.ts:188, apps/core-api/src/modules/discovery/nearby.service.ts:200
- review: sound

### feed: tác giả và hideProfile (c7) with feed [service, in, di-call]

- endpoints: 
- what: FeedController dựng PublicProfileDto cho tác giả qua getByIdOrThrow/findByIds; FeedService gọi PrivacySettingsService.isHidden(authorUserId) trước khi liệt kê bài của người khác.
- why: Bài đăng chỉ lưu authorUserId nên hồ sơ tác giả phải lấy từ user (batch findByIds tránh N+1); hideProfile là cờ của user, nên feed phải hỏi để không lộ bài của người đã ẩn hồ sơ.
- if missing: Post không có tên/avatar tác giả; user bật ẩn hồ sơ vẫn bị xem bài qua feed.
- evidence: apps/core-api/src/modules/feed/feed.controller.ts:208, apps/core-api/src/modules/feed/feed.controller.ts:215, apps/core-api/src/modules/feed/feed.service.ts:153
- review: sound

### friend: hồ sơ danh sách bạn (c8) with friend [service, in, di-call]

- endpoints: 
- what: FriendController.listFriends gọi findByIds để dựng FriendDto kèm PublicProfileDto; nếu thiếu profile thì getByIdOrThrow để ném 404.
- why: Bảng friend chỉ giữ partnerId; tên và avatar của bạn thuộc user nên phải lấy theo lô. Thiếu profile là dữ liệu hỏng nên bị bật thành lỗi thay vì trả danh sách thiếu.
- if missing: Danh sách chat/bạn chỉ có UUID, không có hồ sơ hiển thị.
- evidence: apps/core-api/src/modules/friend/friend.controller.ts:63, apps/core-api/src/modules/friend/friend.controller.ts:73, apps/core-api/src/modules/friend/services/profile-social.service.ts:467
- review: sound

### gift: guest không nhận điểm (c9) with gift [service, in, di-call]

- endpoints: 
- what: GiftService gọi getByIdOrThrow(receiverUserId) để đọc receiver.isGuest trước economy.sendGift và tính pointsAwarded.
- why: Quy tắc docs/06: người nhận là guest vẫn được tặng quà nhưng chân PTS = 0; isGuest nằm ở users nên Gift phải đọc trước khi chuyển số tiền cho Economy.
- if missing: Guest nhận điểm quy đổi, vi phạm quy tắc guest bị giới hạn tính năng.
- evidence: apps/core-api/src/modules/gift/gift.service.ts:201, apps/core-api/src/modules/gift/gift.service.ts:299, apps/core-api/src/modules/gift/gift.service.ts:392
- review: sound

### matching: hồ sơ người mời (c10) with matching [service, in, di-call]

- endpoints: 
- what: InviteController gọi getByIdOrThrow/findByIds để dựng MatchInviteDto kèm hồ sơ inviter cho lời mời ghép.
- why: MatchInvite chỉ lưu inviterUserId; invitee cần thấy ai mời nên matching phải lấy hồ sơ từ user, batch cho danh sách lời mời.
- if missing: Lời mời ghép hiển thị không có thông tin người mời.
- evidence: apps/core-api/src/modules/matching/controllers/invite.controller.ts:107, apps/core-api/src/modules/matching/controllers/invite.controller.ts:138, apps/core-api/src/modules/matching/controllers/invite.controller.ts:145
- review: sound

### party-room: member, region (c11) with party-room [service, in, di-call]

- endpoints: 
- what: PartyRoomService gọi findByIds để dựng hồ sơ thành viên phòng và getByIdOrThrow(hostUserId) lấy host.region để chọn LiveKit URL khi tạo phòng.
- why: Phòng chỉ lưu userId; hồ sơ hiển thị lấy từ user theo lô. Region của host (không phải của người vào) quyết định cụm LiveKit được snapshot cho phòng nên cần đọc lúc tạo.
- if missing: Phòng không hiển thị được thành viên và luôn dùng LiveKit mặc định.
- evidence: apps/core-api/src/modules/party-room/party-room.service.ts:698, apps/core-api/src/modules/party-room/party-room.service.ts:1155
- review: sound

### trừ trust score cùng tx (c12) with safety [service, in, di-call]

- endpoints: 
- what: SafetyService gọi getByIdOrThrow(targetUserId) trước khi report và UserService.adjustTrustScore(manager, userId, -penalty, floor) trong transaction tạo Report.
- why: adjustTrustScore nhận manager của safety để ghi Report và trừ trustScore cùng transaction, UPDATE GREATEST(floor, trust_score + delta) clamp ở sàn nguyên tử. Không có nó, report ghi xong mà điểm không trừ (hoặc ngược lại).
- if missing: Report ghi mà trustScore không trừ (hoặc ngược lại) khi một bước fail; nếu không clamp GREATEST trong cùng UPDATE thì điểm có thể xuống dưới SAFETY_TRUST_SCORE_FLOOR. Chống vượt trần phạt/ngày do khoá hàng user lo.
- evidence: apps/core-api/src/modules/safety/safety.service.ts:66, apps/core-api/src/modules/safety/safety.service.ts:87, apps/core-api/src/modules/safety/safety.service.ts:128
- review: corrected — Trần phạt/ngày và cooldown được chặn bởi khoá hàng user (user.c52); GREATEST chỉ clamp sàn trong một UPDATE nguyên tử, không liên quan số report song song.

### short-video: hồ sơ tác giả (c13) with short-video [service, in, di-call]

- endpoints: 
- what: ShortVideoController dựng PublicProfileDto cho tác giả video qua getByIdOrThrow (một video) và findByIds (cả trang).
- why: Video chỉ lưu authorUserId; thông tin tác giả thuộc user và cần batch cho cả trang để không sinh N truy vấn.
- if missing: Video hiển thị không có tác giả hoặc phát sinh N+1 truy vấn.
- evidence: apps/core-api/src/modules/short-video/short-video.controller.ts:209, apps/core-api/src/modules/short-video/short-video.controller.ts:216
- review: sound

### kiểm ban, mở hồ sơ đối tác (c14) with soul-match [service, in, di-call]

- endpoints: 
- what: SoulMatchService gọi getByIdOrThrow(user.userId) để chặn UserStatus.Banned và getByIdOrThrow(partnerId) để trả PublicProfileDto đối tác sau khi hai bên cùng Thích.
- why: Trạng thái ban nằm ở users; soul-match phải kiểm trước khi vào phòng. Hồ sơ đối tác chỉ được mở khi cả hai cùng thích (PARTNER_LOCKED), nên chỉ gọi user ở nhánh đã đủ điều kiện.
- if missing: User bị khoá vẫn chơi soul-match; không có dữ liệu để mở khoá hồ sơ đối tác.
- evidence: apps/core-api/src/modules/soul-match/soul-match.service.ts:137, apps/core-api/src/modules/soul-match/soul-match.service.ts:420
- review: sound

### gateway ghi presence lease (c50) with signaling-gateway [service, both, redis-state]

- endpoints: e5
- what: ConnectionQuotaService trong signaling-gateway chạy Lua acquire/refresh/release trên key realtimePresenceKey(userId) (ZSET lease theo connection) cùng với key quota.
- why: Gateway là nơi duy nhất biết socket nào đang mở nên chỉ nó ghi presence; user chỉ đọc cùng key (hợp đồng key nằm ở libs/common-dtos realtimePresenceKey) để core-api không phải hỏi gateway qua mạng.
- if missing: Đổi tên key ở một phía làm e5 luôn trả offline mà không có lỗi; nếu core-api tự ghi presence sẽ lệch với số socket thật.
- evidence: apps/signaling-gateway/src/app/connection-quota.service.ts:130
- review: sound

### auth khoá hàng users (c51) with auth [service, in, shared-db]

- endpoints: 
- what: AuthService.linkIdentityAndUpgrade dùng manager.findOne(User, lock pessimistic_write) rồi manager.update/save isGuest=false trong transaction nâng cấp guest.
- why: Nâng cấp guest phải gắn identity, xoá credential guest, đổi isGuest và thu hồi refresh session thành một transaction; khoá hàng users để hai request upgrade song song không cùng thành công.
- if missing: Hai upgrade song song cùng gắn identity, hoặc guest đổi cờ mà refresh session cũ còn sống.
- evidence: apps/core-api/src/modules/auth/auth.service.ts:225
- review: sound

### safety khoá hàng users (c52) with safety [service, in, shared-db]

- endpoints: 
- what: SafetyService trong transaction report gọi manager.getRepository(User).findOne({lock: pessimistic_write}) trên targetUserId trước khi tính cooldown và trần phạt/ngày.
- why: Khoá hàng user bền vững để tuần tự hoá mọi lần phạt trust của cùng một đối tượng; nếu không, hai report song song cùng đọc tổng hôm nay và cùng vượt dailyCap trước khi cái nào thấy cái kia.
- if missing: Trust penalty vượt trần SAFETY_TRUST_PENALTY_DAILY_CAP và cooldown theo cặp bị bỏ qua khi report đồng thời.
- evidence: apps/core-api/src/modules/safety/safety.service.ts:87
- review: sound

### friend khoá hồ sơ để chat (c53) with friend [service, in, shared-db]

- endpoints: 
- what: ProfileSocialService.lockProfileForChatInManager khoá hàng users của profileUserId (pessimistic_write) bằng manager.findOne(User) trước khi mở chat free/gift.
- why: openConversation khoá hàng users của profileUserId để check số first-contact trong ngày (countDailyFirstChatsWithManager vs ngưỡng) và insert conversation không bị tách đôi khi nhiều người nhắn đồng thời; free và gift first-contact cạnh tranh cùng một khoá theo cùng thứ tự nên không deadlock chéo.
- if missing: Nhiều người nhắn song song cùng đọc số first-contact cũ rồi cùng vượt ngưỡng dailyFirstChatCount, hoặc free và gift khoá theo thứ tự khác nhau gây deadlock.
- evidence: apps/core-api/src/modules/friend/services/profile-social.service.ts:467
- review: corrected — Lý do chính theo code là check ngưỡng first-contact/ngày và insert không bị tách đôi; 'duplicate chat' do canonicalPair/unique chứ không chỉ do khoá này.

### matching khoá/đọc users (c54) with matching [service, in, shared-db]

- endpoints: 
- what: GuestMatchQuotaService khoá hàng users (findOneOrFail + pessimistic_write); MatcherWorker và InviteService đọc manager.find(User, In(ids)) trong transaction ghép/nhận lời mời.
- why: GuestMatchQuotaService.authorize khoá hàng users (pessimistic_write) để guest-upgrade của auth và việc join queue có thứ tự rõ ràng; isGuest đọc từ DB chứ không từ JWT nên JWT cũ sau upgrade không giữ quota guest (comment guest-match-quota.service.ts:45-47). Bộ đếm quota không nằm ở users mà ở guest_match_quotas (khoá băm user/device/network, sắp xếp key và insert-before-lock để tránh deadlock/lost update, :83-104). MatcherWorker/InviteService chỉ đọc manager.find(User, In(ids)) trong transaction ghép/nhận lời mời.
- if missing: Guest vào queue song song có thể vượt quota.
- evidence: apps/core-api/src/modules/matching/services/guest-match-quota.service.ts:53
- review: sound

## Notes

- Không có docs/services cho user; mô tả dựa trên code.
- GET /users/:id (getPublicProfile) trả PublicProfileDto cho MỌI user tồn tại: không kiểm hideProfile hay status banned. hideProfile chỉ được áp ở browsePage/findActiveByIds (NOT EXISTS user_privacy_settings.hide_profile), feed (isHidden) và search-by-phone. Doc/ý định 'ẩn hồ sơ' có thể kỳ vọng route này cũng ẩn; cần xác nhận.
- GET /users/:id/presence không kiểm user tồn tại: id lạ trả isOnline=false (getForUser trả mặc định nếu chưa có row privacy), không 404.
- Presence đọc fail-closed: Redis lỗi trả false, không bao giờ báo online sai.
- Nhiều module (auth, safety, friend, matching) đọc/khoá entity User trực tiếp bằng manager dù có UserService; đây là chỗ biên module bị xuyên qua để lấy khoá hàng pessimistic_write cùng transaction.
- docs/06-domain-rules.md:87-91 chỉ yêu cầu hideProfile loại user khỏi Discovery và Feed (và search-by-phone trả null như user không tồn tại); không nêu GET /users/:id. Vì vậy việc route này không kiểm hideProfile/ban là khoảng trống đặc tả chưa rõ chủ ý, chưa đủ để kết luận là bug; soul-match tự kiểm ban ở phía nó (soul-match.service.ts:137).
- banUser/unbanUser (setStatus, user.service.ts:240-256) đọc rồi save entity users bằng manager của admin, không khoá hàng và không thu hồi refresh session của auth; ban chỉ chặn ở lần refresh kế tiếp (xem auth).
