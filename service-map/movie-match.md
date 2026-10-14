# Service map: Movie Match

id: service-map:movie-match
module: module:movie-match
deployable: core-api
confidence: high

Phiên xem chung 2 người, playback state

Sở hữu phiên xem chung 2 người (friend mode: 2 bạn tự chọn URL YouTube; anonymous mode: hàng đợi ghép người lạ, server chọn video, chat + rating, chỉ mutual-like mới mở Friendship) và là nguồn sự thật duy nhất của playback state. KHÔNG stream/proxy video (client tự phát từ URL), KHÔNG có SFU/LiveKit hay tiền, KHÔNG sở hữu quan hệ bạn (Friend) hay block/report (Safety).

Owns: bảng movie_sessions (cặp canonical low/high, videoUrl, positionSeconds, isPlaying, positionUpdatedAt, status, mode friend|anonymous, expiresAt, watchEndedAt, low/highRating, outcome); bảng movie_session_active_participants (PK user_id: 1 user chỉ có 1 phiên active, ghi/xoá cùng transaction với movie_sessions.status); bảng movie_match_queue_entries (PK user_id: hàng chờ ghép ẩn danh riêng của module, không dùng queue của Matching); bảng movie_session_messages (chat ẩn danh append-only, idempotency_key UNIQUE dạng movie:msg:{userId}:{key}, cursor theo seq); Postgres advisory lock 'litmatch:movie-match:pairing' (tuần tự hoá matcher ẩn danh); client ioredis riêng MOVIE_MATCH_REDIS chỉ để PUBLISH realtime:user:{userId}; module không lưu key nào trong Redis

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/movie-match/sessions | jwt | Tạo phiên xem chung với 1 bạn (friendUserId + videoUrl): URL phải http/https và host thuộc MOVIE_MATCH_ALLOWED_VIDEO_HOSTS, bắt buộc areFriends (404 chung), idempotent theo cặp, 409 nếu một trong hai đang active ở phiên khác. |
| e2 | rest | PATCH | /api/v1/movie-match/sessions/:id/state | jwt | Participant cập nhật positionSeconds/isPlaying của phiên bạn bè (last-write-wins, không lock, không Idempotency-Key), 409 nếu đã ended, rồi publish movie.state.changed cho cả hai. |
| e3 | rest | POST | /api/v1/movie-match/sessions/:id/end | jwt | Một trong hai participant kết thúc phiên bạn bè (endReason=left), idempotent; đổi status và xoá 2 dòng active-participant cùng transaction rồi publish movie.session.ended. |
| e4 | rest | GET | /api/v1/movie-match/sessions/:id | jwt | Poll fallback cho realtime: trả MovieSessionDto (kèm partnerUserId) cho participant; không tồn tại hoặc không phải participant đều 404 chung (chống IDOR). |
| e5 | rest | POST | /api/v1/movie-match/anon/queue | jwt | Vào hàng đợi ghép ẩn danh (idempotent): advisory lock tuần tự hoá matcher, ghép với entry cũ nhất vượt qua SafetyService.canPair, server chọn video từ MOVIE_MATCH_ANON_VIDEO_URLS; người đến trước chỉ biết đã ghép qua poll. |
| e6 | rest | GET | /api/v1/movie-match/anon/current | jwt | Trả state phục hồi/poll của flow ẩn danh (idle\|queued\|watching\|rating\|completed) theo vai trò tương đối, không lộ partner; hết giờ xem được xử lý lazy (mở phase rating) ngay trong request này. |
| e7 | rest | DELETE | /api/v1/movie-match/anon/current | jwt | Huỷ hàng đợi hoặc dismiss kết quả/phiên ẩn danh: phiên đang active bị chốt outcome=cancelled cho CẢ HAI và caller được giải phóng active-participant; không publish realtime, bên kia thấy qua poll. |
| e8 | rest | PATCH | /api/v1/movie-match/anon/sessions/:id/state | jwt | Cập nhật playback phiên ẩn danh (last-write-wins) chỉ khi còn trong phần xem (409 sau khi finish/hết giờ), khoá pessimistic_write dòng session rồi publish movie.state.changed. |
| e9 | rest | POST | /api/v1/movie-match/anon/sessions/:id/finish | jwt | Bấm 'Kết thúc' phần xem chung: đặt watchEndedAt và dừng playback để mở phase rating cho cả hai, idempotent; publish movie.state.changed (isPlaying=false). |
| e10 | rest | POST | /api/v1/movie-match/anon/sessions/:id/rating | jwt | Chốt rating like/boring/rude đúng một lần (đổi giá trị → 409, chỉ mở sau watchEndedAt): non-like → not_matched ngay; cả hai like → FriendService.ensureFriendship(source=movie_match) cùng transaction rồi mới lộ partnerUserId. |
| e11 | rest | POST | /api/v1/movie-match/anon/sessions/:id/messages | jwt | Gửi chat ẩn danh trong phiên (HTTP 201): bắt buộc header Idempotency-Key (khoá DB movie:msg:{userId}:{key}, cùng key khác nội dung → 409), giới hạn MOVIE_MATCH_MESSAGE_MAX_LENGTH, không publish realtime (client poll). |
| e12 | rest | GET | /api/v1/movie-match/anon/sessions/:id/messages | jwt | Liệt kê chat của phiên ẩn danh theo cursor seq; chỉ participant đọc được, sender trả vai trò tương đối me\|partner nên không lộ userId. |
| e13 | rest | POST | /api/v1/movie-match/anon/sessions/:id/reactions | jwt | Gửi reaction emoji nổi (whitelist 4 emoji): realtime-only, không persist, không kèm userId, @Throttle 60 request/phút, trả 204 và publish movie.reaction.sent cho cả hai. |
| e14 | socket-emit | EMIT | movie.session.started | internal | Báo cho cả hai bạn có phiên xem chung mới (sessionId, videoUrl, initiatorUserId) sau khi transaction tạo phiên commit; chỉ socket đã verify JWT của đúng người nhận mới nhận. |
| e15 | socket-emit | EMIT | movie.state.changed | internal | Fanout playback state (positionSeconds, isPlaying, positionUpdatedAt) cho cả hai participant, phát từ updateState (friend) và publishAnonPlayback (ẩn danh); chỉ là gợi ý refetch sớm, REST GET vẫn là nguồn sự thật. |
| e16 | socket-emit | EMIT | movie.session.ended | internal | Báo cả hai phiên bạn bè đã kết thúc (reason=left) sau khi endSession commit; nhánh ẩn danh KHÔNG phát event này (kết quả ẩn danh chỉ lấy qua poll). |
| e17 | socket-emit | EMIT | movie.reaction.sent | internal | Fanout reaction emoji ephemeral (sessionId, emoji, không có userId) cho cả hai participant phiên ẩn danh; client lỡ event chỉ mất hiệu ứng. |

## Coordination edges (why this service talks to others)

### bạn bè · mở friendship (c1) with friend [service, out, di-call]

- endpoints: e1, e10
- what: Gọi FriendService.areFriends khi tạo phiên bạn bè; ở rateAnon, khi cả hai 'like' thì gọi FriendService.ensureFriendship(manager, low, high, FriendshipSource.MovieMatch) bằng chính EntityManager của transaction đang giữ khoá phiên.
- why: Friend là chủ của Friendship + Conversation; Movie Match không có bảng quan hệ riêng nên phải hỏi Friend ai là bạn. Mutual-like phải tạo Friendship + Conversation CÙNG transaction với rating để bất biến 'có Friendship ⟺ có Conversation' không lệch (cùng bất biến Soul/Palm Match).
- if missing: Thiếu areFriends: ai cũng ép người lạ vào phiên xem chung và đẩy realtime tới họ. Thiếu ensureFriendship (hoặc gọi ngoài transaction): outcome=matched nhưng không có bạn/hội thoại để chat tiếp, hoặc có Friendship dù rating bị rollback.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:123, apps/core-api/src/modules/movie-match/movie-match.service.ts:510, apps/core-api/src/database/migrations/1755900000000-movie-match-anon.ts:11
- review: sound

### canPair: chặn block/report (c2) with safety [service, out, di-call]

- endpoints: e5
- what: Trong joinAnonQueue, với từng ứng viên (tối đa 20 entry cũ nhất) gọi SafetyService.canPair(userId, candidate.user_id) ngay lúc ghép; ứng viên bị từ chối thì bỏ qua và thử người kế tiếp.
- why: Block/Report thuộc Safety và có thể phát sinh SAU khi user vào hàng đợi; vì ghép với người lạ nên phải hỏi lại đúng lúc ghép (không tin state lúc enqueue), gồm cả cooldown sau report/block gần đây (SAFETY_REMATCH_COOLDOWN_DAYS).
- if missing: Hai người đã block hoặc vừa report nhau vẫn có thể bị ghép lại vào cùng một phiên ẩn danh, vô hiệu hoá lớp bảo vệ của Safety.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:338, apps/core-api/src/modules/safety/safety.service.ts:291, apps/core-api/src/modules/safety/safety.service.ts:298
- review: sound

### phát movie.* sau ghi DB (c3) with signaling-gateway [service, out, redis-pubsub]

- endpoints: e1, e2, e3, e8, e9, e13, e14, e15, e16, e17
- what: Sau khi ghi DB, publishRealtimeEvent đẩy movie.* vào realtime:user:{userId} của CẢ HAI participant; friend (e1,e2,e3) publish sau khi ghi/commit, còn anon e8/e9 gọi publishAnonPlayback ngay trong transaction đang giữ khoá, trước commit. Gateway PSUBSCRIBE rồi emit xuống socket đã xác thực.
- why: Hai participant là hai client khác nhau cần thấy play/pause/seek gần tức thì; core-api không giữ socket nên phải qua gateway (chỉ relay, không business logic). Ẩn danh được quyết ở core-api: payload reaction không có userId vì gateway không đọc/sửa payload.
- if missing: Không có fanout thì client chỉ còn poll (web: 10s phiên bạn bè, 5s phiên ẩn danh): xem chung trễ hàng giây và hiệu ứng reaction (chỉ có qua realtime) mất hẳn; nghiệp vụ trong DB vẫn đúng.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:182, apps/signaling-gateway/src/app/signaling.gateway.ts:332, apps/core-api/src/common/realtime/publish-realtime.ts:12
- review: corrected — anon e8/e9 gọi publishAnonPlayback bên trong transaction (trước commit), không phải sau commit như friend flow.

### PUBLISH realtime:user (c4) with redis [infra, out, redis-pubsub]

- endpoints: e1, e2, e3, e8, e9, e13, e14, e15, e16, e17
- what: Module tạo client ioredis riêng (MOVIE_MATCH_REDIS, từ REDIS_URL) chỉ để PUBLISH lên realtime:user:{userId}; không lưu session, hàng đợi hay khoá nào trong Redis (tất cả nằm ở Postgres).
- why: core-api và signaling-gateway là hai process riêng, không có API nội bộ giữa chúng nên Redis pub/sub là đường duy nhất tới gateway; pub/sub không đảm bảo giao nên state thật phải ghi Postgres trước và client luôn có REST poll.
- if missing: Redis lỗi thì publishRealtimeEvent nuốt lỗi + log warn: nghiệp vụ vẫn thành công nhưng mọi event movie.* im lặng, client chỉ còn polling.
- evidence: apps/core-api/src/modules/movie-match/redis/movie-match-redis.provider.ts:7, apps/core-api/src/modules/movie-match/redis/movie-match-redis.provider.ts:14, apps/signaling-gateway/src/app/signaling.gateway.ts:346
- review: sound

### phiên, hàng đợi, chat (c5) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13
- what: Ghi/đọc movie_sessions, movie_session_active_participants, movie_match_queue_entries, movie_session_messages; dùng pg_advisory_xact_lock cho matcher ẩn danh và khoá dòng pessimistic_write cho các bước đổi phase.
- why: PK user_id của movie_session_active_participants là chốt DB cho 'một user một phiên active' (2 partial index đơn cột không bắt được xung đột chéo low/high); advisory lock chặn hai matcher cùng ghép một ứng viên; khoá dòng làm rating/finish/dismiss tuần tự để quyết mutual-like đúng. Advisory lock là một khoá toàn cục duy nhất cho mọi người dùng (không chia theo region/tuổi), nên ghép ẩn danh tuần tự toàn cluster.
- if missing: Race: một user nằm trong 2 phiên active, hai người cùng ghép một ứng viên, hoặc hai rating đồng thời chốt outcome khác nhau; chat retry sinh tin nhắn trùng.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:153, apps/core-api/src/modules/movie-match/movie-match.service.ts:680, apps/core-api/src/modules/movie-match/entities/movie-session-active-participant.entity.ts:18
- review: sound

### REST + YouTube IFrame (c6) with web [client, in, http]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13
- what: Web gọi REST của module và tự phát video bằng YouTube IFrame API: gửi PATCH state khi user play/pause/seek, nội suy vị trí từ positionSeconds + (now - positionUpdatedAt), poll GET /anon/current mỗi 5s khi queued/watching/rating. Ngoài ra web nghe 4 event movie.* qua socket (xem entry socket riêng).
- why: Server chỉ giữ state, không stream/proxy video nên client phải tự phát và tự nội suy. Flow ẩn danh không publish realtime khi ghép xong, rating, dismiss hay chat mới (chỉ playback + reaction), nên chỉ poll mới thấy các chuyển phase đó.
- if missing: Không có client phát/đồng bộ thì không có video để xem chung; không poll thì người chờ trong hàng đợi không bao giờ biết đã được ghép và bên kia không thấy rating/huỷ.
- evidence: apps/web/src/features/movie-match/components/youtube-player.tsx:120, apps/web/src/features/movie-match/anon-api.ts:18, apps/web/src/features/movie-match/components/movie-session-view.tsx:33
- review: sound

### movie.* → refetch/hiệu ứng (c51) with web [client, out, socket]

- endpoints: e14, e15, e16, e17
- what: Web đăng ký useRealtimeEvent cho movie.session.started (trang movie-match mở phòng khi người kia khởi tạo), movie.state.changed (invalidate/refetch session, cả friend lẫn anon), movie.session.ended (đóng phòng friend) và movie.reaction.sent (hiện emoji nổi).
- why: Hai người xem phải thấy play/pause/seek của nhau gần tức thì nên state được đẩy qua gateway; payload chỉ là gợi ý refetch (REST GET vẫn là nguồn sự thật) trừ reaction là realtime-only không persist nên mất event là mất hiệu ứng. Flow ẩn danh không phát event khi ghép xong/rating/chat nên các chuyển phase đó chỉ lấy được bằng poll.
- if missing: Playback lệch vài giây theo chu kỳ poll (5s anon, 10s friend), người kia không tự được kéo vào phòng khi phiên bạn bè bắt đầu, reaction không bao giờ hiện.
- evidence: apps/web/src/app/(app)/movie-match/page.tsx:31, apps/web/src/features/movie-match/components/movie-session-view.tsx:52, apps/web/src/features/movie-match/components/anon-match-view.tsx:202
- review: sound

### whitelist host YouTube (c52) with external:youtube [external, both, config]

- endpoints: e1, e5
- what: Server chỉ kiểm hostname videoUrl theo MOVIE_MATCH_ALLOWED_VIDEO_HOSTS (mặc định youtube.com,youtu.be, kèm mọi subdomain) và tự chọn video anon từ MOVIE_MATCH_ANON_VIDEO_URLS; web thì chỉ phát được URL mà extractYoutubeVideoId hiểu (youtube.com/watch?v=, youtu.be/) qua YouTube IFrame API.
- why: Server không stream hay proxy video, client tự tải player của YouTube; vì vậy tập host server cho phép phải là tập con của những gì player web phát được. Hai bên sửa độc lập (env vs mã web) nên là một hợp đồng ngầm: server chấp nhận music.youtube.com nhưng web trả null videoId.
- if missing: Thêm host khác hoặc subdomain vào env: phiên tạo thành công (201) nhưng cả hai client không có video để phát; pool anon trỏ tới URL không phải YouTube thì mọi phiên ẩn danh ghép được mà không xem được.
- evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:934, apps/web/src/features/movie-match/youtube-url.ts:17
- review: sound

## Notes

- Doc lỗi thời: docs/services/movie-match-service.md § 1 viết module 'chỉ phụ thuộc friend và realtime-gateway', nhưng code còn import SafetyModule (SafetyService.canPair khi ghép ẩn danh); chính § 9 của doc mới nhắc điều này.
- Doc/comment lỗi thời: § 2-3 của doc, docstring MovieMatchService và entity MovieSession nói 'không có bảng message riêng, chat đi qua Conversation' chỉ đúng cho friend mode; flow ẩn danh có bảng movie_session_messages riêng (2 người chưa là bạn nên chưa có Conversation).
- Doc § 7 chỉ liệt kê 3 event movie.*; code phát thêm movie.reaction.sent (nhánh ẩn danh). docs/services/realtime-gateway.md § 4 (bảng producer) cũng không liệt kê movie-match/mini-game dù § 2 có tên event.
- Doc § 5.5 'không có timer, session không tự hết hạn' chỉ đúng friend mode: phiên ẩn danh có expiresAt nhưng xử lý LAZY khi có request (module không có @Cron/@Interval/@OnEvent nên không có endpoint kind job/consumer). Hết giờ chỉ mở phase rating; MovieMatchOutcome.Expired và MovieSessionEndReason.Replaced có trong enum nhưng không nhánh code nào set.
- Realtime của flow ẩn danh rất mỏng (chỉ 5 call site publishRealtimeEvent trong service): chỉ playback (movie.state.changed) và reaction. joinAnonQueue (ghép xong), rateAnon, dismissAnonCurrent và chat KHÔNG publish; movie.session.ended chỉ phát ở friend mode. Web phải poll GET /anon/current và GET messages mỗi 5s.
- Xác nhận bằng code — lệch quy ước 'publish SAU commit' (docs/03 § 3.7, realtime-gateway.md § 3): publishAnonPlayback được gọi BÊN TRONG callback dataSource.transaction ở updateAnonState (movie-match.service.ts:438) và endAnonWatch (:459) nên event movie.state.changed rời đi trước khi transaction commit (nguy cơ event "ma" nếu commit lỗi, hoặc client refetch trước commit thấy trạng thái cũ); nhánh friend-mode (updateState :209-212, endSession :249-278) publish đúng sau khi ghi xong. Vì publishRealtimeEvent nuốt lỗi và payload chỉ là gợi ý refetch nên hậu quả thực tế nhỏ, nhưng vẫn là vi phạm quy ước.
- Xác nhận bằng đọc code (chưa chạy): getSessionForParticipant (movie-match.service.ts:659-674) chỉ kiểm tra caller là participant, KHÔNG lọc session.mode; MovieSessionDto.from luôn gán partnerUserId (dto/movie-match.dtos.ts:67,79) và friend-mode GET/PATCH/POST end dùng chính hàm đó. Participant của phiên ẨN DANH biết sessionId (anon state view có trả) nên có thể GET /movie-match/sessions/:id để lấy userId đối phương trước mutual-like, phá ẩn danh (trái giả định M5 trong docs/plans/2026-07-15-ui-contract-completion.md). Cũng có thể PATCH /sessions/:id/state (updateState :199, không lock, không lazy-expire) hoặc POST /sessions/:id/end (endSession :244: set status ended + xoá active-participant + phát movie.session.ended) lên phiên ẩn danh, bỏ qua phase rating và outcome của luồng anon.
- NGHI VẤN: friend-mode createSession chỉ gọi areFriends, không gọi SafetyService (block/report); module safety không đụng tới Friendship nên người bị chặn mà vẫn là bạn có thể mở phiên và đẩy realtime tới người chặn. Friend chat thì có check block (friend.service.ts:346). Chưa xác minh bằng chạy thật.
- Mỗi user phải gọi DELETE /anon/current để rời kết quả terminal: completeAnonSession và rateAnon không xoá dòng movie_session_active_participants, và phase rating không có hạn chót; trong lúc đó mở phiên bạn bè bị 409 MOVIE_SESSION_ALREADY_ACTIVE còn vào lại queue chỉ trả state phiên cũ.
- Không có kiểm tra isGuest ở cả friend lẫn anonymous flow (JwtAuthGuard toàn cục cho guest qua); docs/06 chỉ giới hạn guest ở nạp diamond, điểm quy đổi và quota match. Mọi REST endpoint chỉ dùng JWT toàn cục, không có @Public hay @RequireRoles; duy nhất reaction có @Throttle riêng 60/phút.
- Không có inbound DI: slice tĩnh báo inbound=[], chỉ AppModule import MovieMatchModule; index.ts export MovieMatchService 'để dành' theo blueprint. Kỹ thuật queue ghép ẩn danh (advisory lock + PK queue + PK active-participant) được sao chép từ Palm Match, không dùng chung code và không dùng queue Redis của Matching.
- Server không gọi YouTube hay media-server: chỉ whitelist hostname videoUrl bằng new URL() (MOVIE_MATCH_ALLOWED_VIDEO_HOSTS, so khớp hostname === host hoặc hostname.endsWith(`.${host}`) nên nhận cả music.youtube.com/studio.youtube.com); anonymous mode lấy video từ MOVIE_MATCH_ANON_VIDEO_URLS (pool rỗng → 409 ANON_VIDEO_POOL_EMPTY). Web thì chỉ trích videoId từ host youtube.com/youtu.be sau khi bỏ www./m. (youtube-url.ts:15-25), nên URL subdomain khác hoặc host được thêm vào env mà web không hiểu sẽ qua được server nhưng không phát được. Endpoint socket-emit ghi auth internal cho đồng nhất với các module khác (socket nhận event đã qua handshake JWT ở signaling-gateway).
- Kỹ thuật ghép ẩn danh dùng 1 advisory lock DUY NHẤT toàn cục (pg_advisory_xact_lock(hashtext(MOVIE_MATCH_ADVISORY_LOCK_KEY)), movie-match.service.ts:679-683; blocking, không phải try-lock) nên mọi joinAnonQueue/dismiss trên mọi pod tuần tự hoá qua một khoá Postgres, khác Matching (queue Redis chia shard theo matchType/region/ageBand). Đúng cho correctness, nhưng là điểm nghẽn thông lượng khi tính capacity-roadmap.
