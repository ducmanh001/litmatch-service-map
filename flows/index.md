# Cross-service flows (72), grouped

id: flows:index

## Vận hành (admin)

- [admin-ban-and-refund](admin-ban-and-refund.md): Admin khoá tài khoản và hoàn giao dịch
- [admin-broadcast-notification](admin-broadcast-notification.md): Admin gửi thông báo hàng loạt
- [admin-dashboard](admin-dashboard.md): Bảng điều khiển admin
- [admin-economy-inspect-and-config](admin-economy-inspect-and-config.md): Admin xem ví, giao dịch và chỉnh cấu hình kinh tế
- [admin-gift-catalog](admin-gift-catalog.md): Admin quản lý danh mục quà
- [admin-report-resolution](admin-report-resolution.md): Admin xử lý báo cáo vi phạm
- [admin-room-management](admin-room-management.md): Admin xem và đóng phòng party
- [admin-staff-and-permissions](admin-staff-and-permissions.md): Admin quản lý nhân sự và ma trận quyền
- [admin-support-tickets](admin-support-tickets.md): Người dùng gửi ticket hỗ trợ, admin xử lý
- [admin-user-lookup-and-unban](admin-user-lookup-and-unban.md): Admin tra cứu người dùng và mở khoá
- [admin-video-moderation-queue](admin-video-moderation-queue.md): Admin duyệt hàng chờ video
- [app-shell-health-and-readiness](app-shell-health-and-readiness.md): Kiểm tra sẵn sàng của core-api (readiness)

## Danh tính

- [auth-guest-login-and-socket](auth-guest-login-and-socket.md): Đăng nhập khách rồi mở kết nối realtime
- [auth-guest-upgrade](auth-guest-upgrade.md): Nâng cấp tài khoản khách thành tài khoản thật
- [auth-otp-login](auth-otp-login.md): Đăng nhập bằng OTP điện thoại
- [auth-refresh-and-logout](auth-refresh-and-logout.md): Làm mới phiên và đăng xuất (cookie httpOnly + CSRF)
- [auth-search-by-phone](auth-search-by-phone.md): Tìm người theo số điện thoại
- [auth-social-login](auth-social-login.md): Đăng nhập bằng Apple, Google hoặc Facebook
- [user-presence-read](user-presence-read.md): Xem trạng thái online của người dùng

## Tiền tệ & quà

- [avatar-buy-item](avatar-buy-item.md): Mua vật phẩm avatar bằng kim cương
- [economy-outbox-relay](economy-outbox-relay.md): Relay outbox của Economy ra Kafka
- [economy-reconciliation-and-startup-guards](economy-reconciliation-and-startup-guards.md): Đối soát sổ cái và guard khởi động của verifier
- [gift-in-party-room](gift-in-party-room.md): Tặng quà trong phòng party
- [gift-on-video](gift-on-video.md): Tặng quà cho tác giả video
- [gift-to-profile-chat-gate](gift-to-profile-chat-gate.md): Tặng quà từ hồ sơ để mở chat (cổng quà)
- [iap-store-notifications-and-refund-poll](iap-store-notifications-and-refund-poll.md): Hoàn tiền IAP: webhook Apple/Google và job quét refund
- [payos-topup-webhook](payos-topup-webhook.md): Nạp kim cương qua PayOS và webhook
- [vip-purchase-iap](vip-purchase-iap.md): Mua VIP và xác minh biên nhận IAP

## Ghép đôi & gọi

- [calling-end-and-ticker](calling-end-and-ticker.md): Kết thúc cuộc gọi và đồng hồ cuộc gọi
- [calling-like-after-call](calling-like-after-call.md): Like sau cuộc gọi để thành bạn
- [discovery-browse-and-nearby](discovery-browse-and-nearby.md): Khám phá người dùng và gần đây
- [friend-direct-call](friend-direct-call.md): Gọi trực tiếp cho bạn bè
- [matching-invite-accept](matching-invite-accept.md): Chấp nhận lời mời và tạo phiên ghép
- [matching-invite-create](matching-invite-create.md): Mời chơi có chủ đích: tạo và xem lời mời
- [matching-invite-decline-cancel](matching-invite-decline-cancel.md): Từ chối hoặc huỷ lời mời
- [matching-sweeper-jobs](matching-sweeper-jobs.md): Job dọn lời mời và vé ghép đôi hết hạn
- [matching-ticket-status-cancel-speedup](matching-ticket-status-cancel-speedup.md): Xem, huỷ và tăng tốc vé ghép đôi
- [matching-to-call](matching-to-call.md): Từ hàng đợi ghép đôi đến cuộc gọi thoại
- [mini-game-session](mini-game-session.md): Mini game giữa hai bạn bè
- [mood-status-visibility](mood-status-visibility.md): Trạng thái tâm trạng và quyền xem
- [movie-match-anon-chat-finish-reactions](movie-match-anon-chat-finish-reactions.md): Xem phim ẩn danh: chat, reaction và kết thúc
- [movie-match-anonymous](movie-match-anonymous.md): Xem phim chung ẩn danh: hàng đợi, đồng bộ playback, rating
- [movie-match-friend-session](movie-match-friend-session.md): Xem phim chung giữa bạn bè
- [palm-match-queue-and-rating](palm-match-queue-and-rating.md): Palm Match: vào hàng, lật và đánh giá
- [soul-match-room-reads-and-reveal](soul-match-room-reads-and-reveal.md): Đọc phòng Soul Match và lộ hồ sơ sau mutual-like
- [soul-match-send-message](soul-match-send-message.md): Gửi tin nhắn ẩn danh trong phòng Soul Match
- [soul-match-session](soul-match-session.md): Phòng chat ẩn danh Soul Match và mutual-like

## Xã hội & nội dung

- [feed-comment-with-notification](feed-comment-with-notification.md): Bình luận bài kèm thông báo
- [feed-delete-post](feed-delete-post.md): Xoá bài và bình luận
- [feed-post-with-image](feed-post-with-image.md): Đăng bài kèm ảnh: upload presigned rồi gắn asset
- [feed-reaction-with-notification](feed-reaction-with-notification.md): Reaction bài kèm thông báo
- [feed-read-posts](feed-read-posts.md): Đọc feed và bài đăng
- [friend-follow-lists-and-connections](friend-follow-lists-and-connections.md): Danh sách follower/following và danh sách bạn bè
- [friend-list-and-conversation-reads](friend-list-and-conversation-reads.md): Danh sách bạn, đọc tin và streak
- [friend-open-conversation-from-profile](friend-open-conversation-from-profile.md): Mở chat từ hồ sơ trước khi là bạn
- [friend-profile-follow-and-actions](friend-profile-follow-and-actions.md): Theo dõi và xem hành động khả dụng trên hồ sơ
- [friend-streak-warning-job](friend-streak-warning-job.md): Job cảnh báo streak sắp mất
- [message-push-notification](message-push-notification.md): Tin nhắn bạn bè: realtime và Web Push
- [short-video-browse-and-detail](short-video-browse-and-detail.md): Duyệt video ngắn, xem chi tiết, view và reaction
- [short-video-comments](short-video-comments.md): Bình luận video ngắn
- [short-video-publish-moderation](short-video-publish-moderation.md): Đăng video ngắn và duyệt bởi admin
- [short-video-ranking-and-sweeper-jobs](short-video-ranking-and-sweeper-jobs.md): Job tính điểm xếp hạng và dọn video lỗi
- [short-video-report](short-video-report.md): Báo cáo video vi phạm
- [story-create-with-image](story-create-with-image.md): Đăng story kèm ảnh và dọn story hết hạn
- [story-reply-to-dm](story-reply-to-dm.md): Trả lời story thành tin nhắn
- [story-ring-view-and-viewers](story-ring-view-and-viewers.md): Vòng story, xem story và người đã xem

## Phòng party & an toàn

- [party-room-detail-and-comments](party-room-detail-and-comments.md): Xem phòng party và bình luận trong phòng
- [party-room-leave-and-role-change](party-room-leave-and-role-change.md): Rời phòng và đổi vai trò thành viên
- [party-room-lifecycle](party-room-lifecycle.md): Vòng đời phòng party với LiveKit
- [party-room-speaker-invite](party-room-speaker-invite.md): Mời lên nói và phản hồi lời mời
- [safety-block-and-unblock](safety-block-and-unblock.md): Chặn và bỏ chặn người dùng
- [safety-report-and-trust-score](safety-report-and-trust-score.md): Báo cáo người dùng và trừ điểm tin cậy

## Coverage

{"total":214,"flow":188,"local":26,"cross-open":0,"unverified":0}

Entrypoints confirmed as local-only (no cross-service coordination):
- user: GET /users/me — Đọc một dòng users bằng userRepo.findOneBy trong UserService.getByIdOrThrow, 404 nếu không có. Chỉ bảng của module user…
- user: PATCH /users/me — Đọc User theo id rồi gộp các trường hồ sơ, kiểm tra tuổi/khoảng seeking thuần hàm và userRepo.save vào bảng user của ch…
- user: GET /users/me/privacy — Đọc một dòng privacy_settings của user (findOneBy) rồi dựng DTO với DEFAULT_PRIVACY_SETTINGS nếu chưa có. Chỉ bảng của …
- user: PUT /users/me/privacy — Chỉ upsert bảng PrivacySetting của user theo userId rồi đọc lại bằng findOneBy; không gọi module/provider khác, không p…
- user: GET /users/:id — Chỉ findOneBy User theo id rồi map PublicProfileDto; chỉ đọc bảng của user.
- app-shell: GET /health — Alias gọi live() trả { status: 'ok', uptimeSeconds } từ process.uptime(); không chạm DB/Redis/dependency.
- app-shell: GET /health/live — Handler thuần trả status ok và uptime từ process.uptime(); không gọi ReadinessService, Postgres hay Redis.
- app-shell: GET /capabilities — Chỉ suy ra CapabilitiesDto từ ConfigService (env) và danh sách bảo trì trong bộ nhớ; không gọi provider thật, DB hay Re…
- economy: GET /economy/wallet — Chỉ walletRepo.findOneBy Wallet của user trong DB economy, trả 0 nếu chưa có ví; không ghi ledger hay gọi service khác.
- economy: GET /economy/iap/products — Chỉ productRepo.find IapProduct active sắp theo diamonds trong DB economy; không verify receipt với Apple/Google.
- economy: GET /economy/payos/packages — Chỉ packageRepo.find các PayosPackage active theo amountVnd tăng dần trong DB economy rồi map view; không gọi payOS.
- economy: GET /economy/vip/plans — Chỉ planRepo.find VipPlan active sắp theo priceDiamond trong DB economy, rồi map view.
- economy: GET /economy/transactions — Chỉ SELECT ledger_transactions/ledger_entries/ledger_accounts của economy với cursor pagination; read-only, không ghi l…
- mini-game: POST /mini-game/sessions/:id/cancel — Chỉ đọc MiniGameSession qua sessionRepo rồi trong một transaction update status=Cancelled và xoá MiniGameActiveParticip…
- mini-game: GET /mini-game/sessions/:id — Chỉ đọc một dòng mini_game_sessions bằng sessionRepo.findOneBy rồi kiểm tra participant; 404 gộp nếu không phải thành v…
- notification: GET /notifications — Query builder trên bảng notifications của user, phân trang cursor theo seq, giải mã cursor bằng helper thuần. Chỉ đọc D…
- notification: GET /notifications/unread-count — Chỉ đếm Notification có readAt IS NULL của user bằng query builder trên bảng của notification; read-only.
- notification: DELETE /notifications/web-push/subscription — Chỉ xoá hàng WebPushSubscription theo userId+endpoint trong DB của notification; không gửi push hay gọi provider.
- notification: POST /notifications/:notificationId/read — Đọc bảng notifications theo id, kiểm tra chủ sở hữu rồi update readAt; idempotent. Chỉ chạm bảng của module notificatio…
- avatar: GET /avatar/catalog — Chỉ find AvatarAsset active theo slot tuỳ chọn, sắp sortOrder/code; read-only trên bảng của avatar, không gọi EconomySe…
- avatar: GET /avatar/me/items — Đọc user_avatar_items của user rồi avatar_assets theo id (findBy + find). Chỉ bảng của module avatar, không gọi Economy…
- avatar: GET /avatar/me — Đọc user_avatar_config và avatar_assets của chính module avatar; lần đầu lazy-init ghi user_avatar_items + user_avatar_…
- avatar: GET /avatar/users/:userId — Chỉ đọc user_avatar_config và avatar_assets (resolveLayers) của module avatar; chưa có config thì trả layers rỗng tại c…
- signaling-gateway: GET /health — Liveness thuần: trả status ok và process.uptime(), không chạm Redis hay service nào.
- signaling-gateway: GET /health/live — Liveness chỉ trả { status: ok, uptimeSeconds } từ process.uptime(), không chạm Redis, DB hay service nào khác.
- signaling-gateway: GET /health/ready — Chỉ đọc cờ in-memory signaling.isReady() và redisAdapter.isReady() của chính gateway rồi trả 200/503; không gọi I/O hay…
