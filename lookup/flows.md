# Lookup: cross-service flows

id: lookup:flows

Flow id to its file.

| id | file | note |
|---|---|---|
| flow:admin-ban-and-refund | ../flows/admin-ban-and-refund.md | Admin khoá một người dùng và hoàn một giao dịch diamond. AdminModule chỉ điều phối: dữ liệu users do user sở … |
| flow:admin-broadcast-notification | ../flows/admin-broadcast-notification.md | Admin broadcast thông báo tới mọi user active: Admin điều phối, User cấp danh sách người nhận, Economy cấp da… |
| flow:admin-dashboard | ../flows/admin-dashboard.md | Admin mở dashboard: dữ liệu nằm ở ba module khác nhau (người dùng, kinh tế, phòng party) cộng nhật ký audit, … |
| flow:admin-economy-inspect-and-config | ../flows/admin-economy-inspect-and-config.md | Admin xem ví, lịch sử giao dịch và catalog kinh tế rồi bật/tắt IAP product hoặc VIP plan; Admin chỉ điều phối… |
| flow:admin-gift-catalog | ../flows/admin-gift-catalog.md | Admin xem, tạo và sửa quà trong catalog; giá kim cương là dữ liệu trong bảng gifts do Gift sở hữu, còn Admin … |
| flow:admin-report-resolution | ../flows/admin-report-resolution.md | Admin xem hàng đợi báo cáo rồi đánh dấu đã xử lý hoặc bác bỏ. AdminModule kiểm quyền và điều phối, SafetyModu… |
| flow:admin-room-management | ../flows/admin-room-management.md | Admin liệt kê phòng party đang live rồi đóng một phòng. Việc này chạm nhiều dịch vụ: AdminModule kiểm quyền v… |
| flow:admin-staff-and-permissions | ../flows/admin-staff-and-permissions.md | Admin xem và bật/tắt quyền theo vai trò, liệt kê nhân sự và đổi vai trò. AdminModule giữ ma trận quyền và aud… |
| flow:admin-support-tickets | ../flows/admin-support-tickets.md | Người dùng gửi ticket kèm Idempotency-Key để không bị tạo trùng khi gọi lại, rồi xem lại ticket của mình. Sup… |
| flow:admin-user-lookup-and-unban | ../flows/admin-user-lookup-and-unban.md | Admin xem danh sách, xem chi tiết rồi mở khoá một tài khoản. Module Admin chỉ điều phối: dữ liệu users thuộc … |
| flow:admin-video-moderation-queue | ../flows/admin-video-moderation-queue.md | Moderator xem video chờ duyệt và video đã đăng, rồi từ chối hoặc gỡ video. AdminModule kiểm quyền và ghi audi… |
| flow:app-shell-health-and-readiness | ../flows/app-shell-health-and-readiness.md | Probe Docker/k8s gọi /health/ready để biết core-api có nhận traffic được không; app-shell phải hỏi thật Postg… |
| flow:auth-guest-login-and-socket | ../flows/auth-guest-login-and-socket.md | Người dùng mới lấy phiên khách từ core-api (auth + user + PostgreSQL), rồi mở Socket.IO tới signaling-gateway… |
| flow:auth-guest-upgrade | ../flows/auth-guest-upgrade.md | Người dùng đang là guest gắn danh tính OTP điện thoại hoặc social (Google/Apple/Facebook) vào chính userId hi… |
| flow:auth-otp-login | ../flows/auth-otp-login.md | Người dùng xin mã OTP rồi xác minh để nhận phiên đăng nhập. Auth giữ danh tính (auth_identities), OTP và refr… |
| flow:auth-refresh-and-logout | ../flows/auth-refresh-and-logout.md | Web giữ refresh token trong cookie httpOnly; khi access token hết hạn web gọi /auth/refresh để xoay vòng toke… |
| flow:auth-search-by-phone | ../flows/auth-search-by-phone.md | Người dùng đã đăng nhập tìm hồ sơ người khác bằng số điện thoại. Module auth giữ danh tính điện thoại còn mod… |
| flow:auth-social-login | ../flows/auth-social-login.md | Client gửi ID token (Google/Apple) hoặc access token (Facebook) lên POST /auth/social. Server không tin token… |
| flow:avatar-buy-item | ../flows/avatar-buy-item.md | Người dùng nhận item miễn phí, mua item trả phí bằng DIA với Idempotency-Key rồi trang bị. Avatar sở hữu cata… |
| flow:calling-end-and-ticker | ../flows/calling-end-and-ticker.md | Một cuộc gọi Voice Match có thể kết thúc từ nhiều phía (poll GET, nút kết thúc, ticker nền). Calling chốt Cal… |
| flow:calling-like-after-call | ../flows/calling-like-after-call.md | Sau Voice Match, mỗi bên bấm like; khi cả hai đã like thì Calling ghi reaction và nhờ Friend mở Friendship cù… |
| flow:discovery-browse-and-nearby | ../flows/discovery-browse-and-nearby.md | Người dùng duyệt danh sách theo giới tính/tuổi và tìm người gần bằng vị trí đã quantize. Discovery sở hữu vị … |
| flow:economy-outbox-relay | ../flows/economy-outbox-relay.md | Economy ghi OutboxEvent cùng giao dịch sổ cái; một job nền định kỳ đọc các dòng chưa gửi, publish lên Kafka r… |
| flow:economy-reconciliation-and-startup-guards | ../flows/economy-reconciliation-and-startup-guards.md | Khi core-api khởi động, economy đăng ký hai job đối soát chỉ đọc (fast và deep) để tự phát hiện lệch Nợ/Có, v… |
| flow:feed-comment-with-notification | ../flows/feed-comment-with-notification.md | Người dùng bình luận lên bài đăng: feed phải hỏi safety (block hai chiều), friend (audience bạn bè) và user (… |
| flow:feed-delete-post | ../flows/feed-delete-post.md | Người dùng xoá mềm bài viết hoặc bình luận của chính mình trong feed; feed phải hỏi safety, friend, user về q… |
| flow:feed-post-with-image | ../flows/feed-post-with-image.md | Client không gửi file qua API: xin presigned URL từ media, upload thẳng lên object storage, rồi đăng bài chỉ … |
| flow:feed-reaction-with-notification | ../flows/feed-reaction-with-notification.md | Người dùng thả tim, bỏ tim và xem trạng thái reaction của một bài đăng. Feed sở hữu Reaction nhưng quyền xem … |
| flow:feed-read-posts | ../flows/feed-read-posts.md | Người dùng đọc feed công khai, timeline của một tác giả và chi tiết một bài. Feed chỉ sở hữu Post, nên mọi qu… |
| flow:friend-direct-call | ../flows/friend-direct-call.md | Hai người đã follow nhau gọi voice lâu dài không qua ghép đôi: Calling phải hỏi Friend (và Safety bên trong F… |
| flow:friend-follow-lists-and-connections | ../flows/friend-follow-lists-and-connections.md | Trang hồ sơ và trang Bạn bè đọc số follow, danh sách follower/following theo cursor và danh sách bạn bè (ghép… |
| flow:friend-list-and-conversation-reads | ../flows/friend-list-and-conversation-reads.md | Màn chat cần danh sách hội thoại kèm hồ sơ đối phương, lịch sử tin nhắn, streak, mốc đã đọc và trạng thái tắt… |
| flow:friend-open-conversation-from-profile | ../flows/friend-open-conversation-from-profile.md | Người xem bấm nhắn tin trên hồ sơ của người lạ. friend phải hỏi user (hồ sơ có tồn tại) và safety (có block k… |
| flow:friend-profile-follow-and-actions | ../flows/friend-profile-follow-and-actions.md | Khi người dùng mở hồ sơ người khác, module friend phải xác nhận hồ sơ tồn tại (user) và không bị block hai ch… |
| flow:friend-streak-warning-job | ../flows/friend-streak-warning-job.md | Job nền của module friend chạy theo chu kỳ, chỉ một replica được chạy nhờ advisory lock trong Postgres. Nó tì… |
| flow:gift-in-party-room | ../flows/gift-in-party-room.md | Người dùng tặng quà trong phòng party đang diễn ra. Gift giữ catalog và GiftEvent, Economy là nơi duy nhất gh… |
| flow:gift-on-video | ../flows/gift-on-video.md | Người xem tặng quà cho tác giả video: Gift tra catalog và người nhận, Economy trừ DIA người tặng và cộng PTS … |
| flow:gift-to-profile-chat-gate | ../flows/gift-to-profile-chat-gate.md | Người dùng tặng quà từ profile người khác để mở chat 1-1. Gift tính điểm, Economy ghi sổ kép, Friend tạo conv… |
| flow:iap-store-notifications-and-refund-poll | ../flows/iap-store-notifications-and-refund-poll.md | Khi Apple/Google hoàn tiền một giao dịch IAP, economy phải xác minh chữ ký webhook, rồi ghi bút toán đảo vào … |
| flow:matching-invite-accept | ../flows/matching-invite-accept.md | Invitee chấp nhận lời mời ghép: matching phải khoá invite, nhờ safety kiểm tra block/báo cáo, đọc trạng thái … |
| flow:matching-invite-create | ../flows/matching-invite-create.md | Người dùng mời trực tiếp một người khác chơi Voice/Soul Match. Module matching sở hữu MatchInvite nhưng phải … |
| flow:matching-invite-decline-cancel | ../flows/matching-invite-decline-cancel.md | Invitee từ chối hoặc inviter huỷ một MatchInvite. matching khoá dòng invite và kiểm tra hết hạn tại thời điểm… |
| flow:matching-sweeper-jobs | ../flows/matching-sweeper-jobs.md | Hai worker nền của matching chạy theo timer trên mọi replica, nên phải dùng advisory lock Postgres để chỉ một… |
| flow:matching-ticket-status-cancel-speedup | ../flows/matching-ticket-status-cancel-speedup.md | Người dùng đọc trạng thái vé và giá tăng tốc theo hạng VIP, huỷ vé đang xếp hàng, hoặc trả DIA để tăng ưu tiê… |
| flow:matching-to-call | ../flows/matching-to-call.md | Hai người xếp hàng ghép đôi, worker ghép cặp và hai bên xác nhận, sau đó vào cuộc gọi LiveKit. Matching sở hữ… |
| flow:message-push-notification | ../flows/message-push-notification.md | Một người gửi tin nhắn 1-1; module friend lưu tin, cập nhật streak, rồi báo cho người nhận qua realtime (Redi… |
| flow:mini-game-session | ../flows/mini-game-session.md | Một bạn mời bạn bè chơi oẳn tù tì, hai bên nộp move ẩn theo lượt; mini-game phải hỏi friend xem hai người có … |
| flow:mood-status-visibility | ../flows/mood-status-visibility.md | Người dùng đặt và xoá mood theo preset với Idempotency-Key, rồi người khác xem mood đó. Mood sở hữu log sự ki… |
| flow:movie-match-anon-chat-finish-reactions | ../flows/movie-match-anon-chat-finish-reactions.md | Trong phiên xem phim ẩn danh, hai người chat có Idempotency-Key, gửi reaction tạm thời, bấm kết thúc và dọn t… |
| flow:movie-match-anonymous | ../flows/movie-match-anonymous.md | Người lạ xếp hàng xem phim chung. movie-match ghép cặp (hỏi safety có được ghép không), đồng bộ playback qua … |
| flow:movie-match-friend-session | ../flows/movie-match-friend-session.md | Hai người bạn xem chung một video YouTube: movie-match phải kiểm tra quan hệ bạn bè với friend, ghi phiên và … |
| flow:palm-match-queue-and-rating | ../flows/palm-match-queue-and-rating.md | Người dùng vào hàng đợi ghép cặp ẩn danh, lật bài rồi like/skip. Palm Match sở hữu hàng đợi và phiên nhưng kh… |
| flow:party-room-detail-and-comments | ../flows/party-room-detail-and-comments.md | Client xem danh sách phòng, chi tiết phòng (kèm hồ sơ member lấy từ module user), đọc comment rồi gửi comment… |
| flow:party-room-leave-and-role-change | ../flows/party-room-leave-and-role-change.md | Member rời phòng party hoặc host thu quyền speaker: party-room đổi trạng thái trong PostgreSQL dưới lock phòn… |
| flow:party-room-lifecycle | ../flows/party-room-lifecycle.md | Phòng party có DB (party-room) là nguồn sự thật về thành viên, còn âm thanh chạy qua LiveKit SFU (media-serve… |
| flow:party-room-speaker-invite | ../flows/party-room-speaker-invite.md | Host mời một khán giả lên speaker; khán giả chấp nhận hoặc từ chối. Chấp nhận phải đổi role trong PostgreSQL … |
| flow:payos-topup-webhook | ../flows/payos-topup-webhook.md | Người dùng tạo đơn nạp PayOS rồi thanh toán trên trang của PayOS; tiền chỉ được cộng vào ví khi webhook có ch… |
| flow:safety-block-and-unblock | ../flows/safety-block-and-unblock.md | Người dùng chặn, bỏ chặn hoặc kiểm tra trạng thái chặn một người khác. Safety sở hữu log Block append-only nh… |
| flow:safety-report-and-trust-score | ../flows/safety-report-and-trust-score.md | Khi một người dùng báo cáo người khác, safety phải ghi Report append-only và trừ trust score của người bị báo… |
| flow:short-video-browse-and-detail | ../flows/short-video-browse-and-detail.md | Người dùng duyệt danh sách video published (recent hoặc ranked, hoặc tab following), mở chi tiết, ghi watch-t… |
| flow:short-video-comments | ../flows/short-video-comments.md | Người dùng thêm, xem và xoá comment của video ngắn. short-video sở hữu video và comment (kèm commentCount), n… |
| flow:short-video-publish-moderation | ../flows/short-video-publish-moderation.md | Người dùng xin URL upload, upload video rồi finalize; tuỳ VIDEO_MODERATION_MODE video sang pending_review hoặ… |
| flow:short-video-ranking-and-sweeper-jobs | ../flows/short-video-ranking-and-sweeper-jobs.md | Hai worker nền của short-video chạy theo setInterval: một job tính lại rank_score cho video published, một jo… |
| flow:short-video-report | ../flows/short-video-report.md | Người xem báo cáo một video ngắn; short-video phải xác nhận video tồn tại và hiển thị được trước, Safety ghi … |
| flow:soul-match-room-reads-and-reveal | ../flows/soul-match-room-reads-and-reveal.md | Phòng chat ẩn danh Soul Match thuộc soul-match nhưng MatchSession do matching sở hữu và Friendship do friend … |
| flow:soul-match-send-message | ../flows/soul-match-send-message.md | Thành viên gửi tin nhắn ẩn danh trong phòng Soul Match. Soul Match không sở hữu MatchSession và quan hệ bạn b… |
| flow:soul-match-session | ../flows/soul-match-session.md | Hai người đã được Matching ghép đôi chat ẩn danh rồi đánh giá lẫn nhau; khi cả hai like thì Soul Match phải n… |
| flow:story-create-with-image | ../flows/story-create-with-image.md | Story chỉ được lưu khi ảnh đã được Media xác nhận là của đúng chủ, đúng purpose Story và đã upload hợp lệ; sa… |
| flow:story-reply-to-dm | ../flows/story-reply-to-dm.md | Người xem trả lời một story: Feed không có chat riêng nên phải hỏi Safety/Friend/User xem có được xem story k… |
| flow:story-ring-view-and-viewers | ../flows/story-ring-view-and-viewers.md | Feed sở hữu Story/StoryView nhưng không sở hữu quan hệ bạn bè, block hay quyền riêng tư, nên mỗi lần liệt kê … |
| flow:user-presence-read | ../flows/user-presence-read.md | Khi một người dùng xem trạng thái online của người khác, User đọc cài đặt riêng tư trong PostgreSQL rồi đọc k… |
| flow:vip-purchase-iap | ../flows/vip-purchase-iap.md | Hai luồng tiền chạy qua cùng một sổ cái: mua VIP bằng diamond với Idempotency-Key, và nạp diamond qua IAP chỉ… |
