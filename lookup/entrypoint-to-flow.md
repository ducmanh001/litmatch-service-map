# Lookup: entrypoint to flow

id: lookup:entrypoint-to-flow

Every REST endpoint, webhook, job and consumer of the service map and the cross-service flow(s) that cover it. "local" means the entrypoint is handled inside one service.

| service | entrypoint | kind | status | flows |
|---|---|---|---|---|
| auth | POST /auth/guest | rest | flow | ../flows/auth-guest-login-and-socket.md |
| auth | POST /auth/otp/request | rest | flow | ../flows/auth-otp-login.md |
| auth | POST /auth/otp/verify | rest | flow | ../flows/auth-otp-login.md |
| auth | POST /auth/social | rest | flow | ../flows/auth-social-login.md |
| auth | POST /auth/refresh | rest | flow | ../flows/auth-refresh-and-logout.md |
| auth | POST /auth/logout | rest | flow | ../flows/auth-refresh-and-logout.md |
| auth | POST /auth/upgrade/otp | rest | flow | ../flows/auth-guest-upgrade.md |
| auth | POST /auth/upgrade/social | rest | flow | ../flows/auth-guest-upgrade.md |
| auth | GET /auth/search-by-phone | rest | flow | ../flows/auth-search-by-phone.md |
| user | GET /users/me | rest | local |  |
| user | PATCH /users/me | rest | local |  |
| user | GET /users/me/privacy | rest | local |  |
| user | PUT /users/me/privacy | rest | local |  |
| user | GET /users/:id/presence | rest | flow | ../flows/user-presence-read.md |
| user | GET /users/:id | rest | local |  |
| support | POST /support/tickets | rest | flow | ../flows/admin-support-tickets.md |
| support | GET /support/tickets/me | rest | flow | ../flows/admin-support-tickets.md |
| app-shell | GET /health | rest | local |  |
| app-shell | GET /health/live | rest | local |  |
| app-shell | GET /health/ready | rest | flow | ../flows/app-shell-health-and-readiness.md |
| app-shell | GET /capabilities | rest | local |  |
| admin | GET /admin/dashboard | rest | flow | ../flows/admin-dashboard.md |
| admin | GET /admin/users | rest | flow | ../flows/admin-user-lookup-and-unban.md |
| admin | GET /admin/users/:id | rest | flow | ../flows/admin-user-lookup-and-unban.md |
| admin | POST /admin/users/:id/ban | rest | flow | ../flows/admin-ban-and-refund.md |
| admin | POST /admin/users/:id/unban | rest | flow | ../flows/admin-user-lookup-and-unban.md |
| admin | GET /admin/reports | rest | flow | ../flows/admin-report-resolution.md |
| admin | POST /admin/reports/:id/resolve | rest | flow | ../flows/admin-report-resolution.md |
| admin | POST /admin/reports/:id/dismiss | rest | flow | ../flows/admin-report-resolution.md |
| admin | GET /admin/videos/pending | rest | flow | ../flows/admin-video-moderation-queue.md |
| admin | GET /admin/videos/published | rest | flow | ../flows/admin-video-moderation-queue.md |
| admin | POST /admin/videos/:id/approve | rest | flow | ../flows/short-video-publish-moderation.md |
| admin | POST /admin/videos/:id/reject | rest | flow | ../flows/admin-video-moderation-queue.md |
| admin | POST /admin/videos/:id/remove | rest | flow | ../flows/admin-video-moderation-queue.md |
| admin | POST /admin/rooms/:id/close | rest | flow | ../flows/admin-room-management.md |
| admin | GET /admin/rooms | rest | flow | ../flows/admin-room-management.md |
| admin | GET /admin/support/tickets | rest | flow | ../flows/admin-support-tickets.md |
| admin | PATCH /admin/support/tickets/:id | rest | flow | ../flows/admin-support-tickets.md |
| admin | GET /admin/gifts | rest | flow | ../flows/admin-gift-catalog.md |
| admin | POST /admin/gifts | rest | flow | ../flows/admin-gift-catalog.md |
| admin | PATCH /admin/gifts/:id | rest | flow | ../flows/admin-gift-catalog.md |
| admin | GET /admin/economy/wallet/:userId | rest | flow | ../flows/admin-economy-inspect-and-config.md |
| admin | GET /admin/economy/users/:userId/transactions | rest | flow | ../flows/admin-economy-inspect-and-config.md |
| admin | POST /admin/economy/transactions/:id/refund | rest | flow | ../flows/admin-ban-and-refund.md |
| admin | GET /admin/config/economy-catalog | rest | flow | ../flows/admin-economy-inspect-and-config.md |
| admin | PATCH /admin/config/iap-products/:productId | rest | flow | ../flows/admin-economy-inspect-and-config.md |
| admin | PATCH /admin/config/vip-plans/:id | rest | flow | ../flows/admin-economy-inspect-and-config.md |
| admin | POST /admin/notifications/broadcast | rest | flow | ../flows/admin-broadcast-notification.md |
| admin | GET /admin/permissions | rest | flow | ../flows/admin-staff-and-permissions.md |
| admin | PATCH /admin/permissions/:role/:permission | rest | flow | ../flows/admin-staff-and-permissions.md |
| admin | GET /admin/staff | rest | flow | ../flows/admin-staff-and-permissions.md |
| admin | PATCH /admin/staff/:id/role | rest | flow | ../flows/admin-staff-and-permissions.md |
| economy | GET /economy/wallet | rest | local |  |
| economy | GET /economy/iap/products | rest | local |  |
| economy | GET /economy/payos/packages | rest | local |  |
| economy | POST /economy/payos/orders | rest | flow | ../flows/payos-topup-webhook.md |
| economy | GET /economy/payos/orders/:orderId | rest | flow | ../flows/payos-topup-webhook.md |
| economy | GET /economy/vip/plans | rest | local |  |
| economy | POST /economy/iap/verify | rest | flow | ../flows/vip-purchase-iap.md |
| economy | POST /economy/vip/purchase | rest | flow | ../flows/vip-purchase-iap.md |
| economy | GET /economy/transactions | rest | local |  |
| economy | POST /economy/webhooks/apple | webhook | flow | ../flows/iap-store-notifications-and-refund-poll.md |
| economy | POST /economy/webhooks/google/rtdn | webhook | flow | ../flows/iap-store-notifications-and-refund-poll.md |
| economy | POST /economy/webhooks/payos | webhook | flow | ../flows/payos-topup-webhook.md |
| economy | IapRefundPollService.onApplicationBootstrap | job | flow | ../flows/iap-store-notifications-and-refund-poll.md |
| economy | OutboxRelayService.onApplicationBootstrap | job | flow | ../flows/economy-outbox-relay.md |
| economy | ReconciliationService.onApplicationBootstrap | job | flow | ../flows/economy-reconciliation-and-startup-guards.md |
| economy | DevIapVerifier.onApplicationBootstrap | job | flow | ../flows/economy-reconciliation-and-startup-guards.md |
| economy | DevAppleNotificationVerifier.onApplicationBootstrap | job | flow | ../flows/economy-reconciliation-and-startup-guards.md |
| economy | DevGoogleRtdnVerifier.onApplicationBootstrap | job | flow | ../flows/economy-reconciliation-and-startup-guards.md |
| gift | GET /gifts | rest | flow | ../flows/admin-gift-catalog.md |
| gift | POST /party/rooms/:roomId/gifts | rest | flow | ../flows/gift-in-party-room.md |
| gift | POST /videos/:videoId/gifts | rest | flow | ../flows/gift-on-video.md |
| gift | POST /profiles/:profileUserId/gifts | rest | flow | ../flows/gift-to-profile-chat-gate.md |
| matching | POST /matching/invites | rest | flow | ../flows/matching-invite-create.md |
| matching | GET /matching/invites | rest | flow | ../flows/matching-invite-create.md |
| matching | GET /matching/invites/:id | rest | flow | ../flows/matching-invite-create.md |
| matching | POST /matching/invites/:id/accept | rest | flow | ../flows/matching-invite-accept.md |
| matching | POST /matching/invites/:id/decline | rest | flow | ../flows/matching-invite-decline-cancel.md |
| matching | POST /matching/invites/:id/cancel | rest | flow | ../flows/matching-invite-decline-cancel.md |
| matching | POST /matching/tickets | rest | flow | ../flows/matching-to-call.md |
| matching | GET /matching/tickets/current | rest | flow | ../flows/matching-ticket-status-cancel-speedup.md |
| matching | GET /matching/tickets/:id | rest | flow | ../flows/matching-ticket-status-cancel-speedup.md |
| matching | DELETE /matching/tickets/:id | rest | flow | ../flows/matching-ticket-status-cancel-speedup.md |
| matching | POST /matching/tickets/:id/confirm | rest | flow | ../flows/matching-to-call.md |
| matching | POST /matching/tickets/:id/speedup | rest | flow | ../flows/matching-ticket-status-cancel-speedup.md |
| matching | InviteSweeperService.onApplicationBootstrap | job | flow | ../flows/matching-sweeper-jobs.md |
| matching | MatcherWorkerService.onApplicationBootstrap | job | flow | ../flows/matching-to-call.md |
| matching | TicketSweeperService.onApplicationBootstrap | job | flow | ../flows/matching-sweeper-jobs.md |
| calling | POST /calling/match-sessions/:matchSessionId/join | rest | flow | ../flows/matching-to-call.md |
| calling | POST /calling/friends/:friendUserId/join | rest | flow | ../flows/friend-direct-call.md |
| calling | GET /calling/calls/:id | rest | flow | ../flows/calling-end-and-ticker.md |
| calling | POST /calling/calls/:id/end | rest | flow | ../flows/matching-to-call.md |
| calling | POST /calling/match-sessions/:matchSessionId/end | rest | flow | ../flows/calling-end-and-ticker.md |
| calling | POST /calling/calls/:id/like | rest | flow | ../flows/calling-like-after-call.md |
| calling | POST /calling/webhooks/livekit | webhook | flow | ../flows/matching-to-call.md |
| calling | CallTickerService.onApplicationBootstrap | job | flow | ../flows/calling-end-and-ticker.md |
| soul-match | GET /soul-match/sessions/:id | rest | flow | ../flows/soul-match-room-reads-and-reveal.md |
| soul-match | GET /soul-match/sessions/:id/messages | rest | flow | ../flows/soul-match-room-reads-and-reveal.md |
| soul-match | POST /soul-match/sessions/:id/messages | rest | flow | ../flows/soul-match-send-message.md |
| soul-match | POST /soul-match/sessions/:id/rating | rest | flow | ../flows/soul-match-session.md |
| soul-match | POST /soul-match/sessions/:id/end | rest | flow | ../flows/soul-match-session.md |
| soul-match | GET /soul-match/sessions/:id/partner | rest | flow | ../flows/soul-match-room-reads-and-reveal.md |
| palm-match | GET /palm-match/reading | rest | flow | ../flows/palm-match-queue-and-rating.md |
| palm-match | POST /palm-match/queue | rest | flow | ../flows/palm-match-queue-and-rating.md |
| palm-match | GET /palm-match/current | rest | flow | ../flows/palm-match-queue-and-rating.md |
| palm-match | POST /palm-match/sessions/:id/flip | rest | flow | ../flows/palm-match-queue-and-rating.md |
| palm-match | POST /palm-match/sessions/:id/rating | rest | flow | ../flows/palm-match-queue-and-rating.md |
| palm-match | DELETE /palm-match/current | rest | flow | ../flows/palm-match-queue-and-rating.md |
| mood | GET /mood/presets | rest | flow | ../flows/mood-status-visibility.md |
| mood | GET /mood/status/me | rest | flow | ../flows/mood-status-visibility.md |
| mood | GET /mood/status/:userId | rest | flow | ../flows/mood-status-visibility.md |
| mood | POST /mood/status | rest | flow | ../flows/mood-status-visibility.md |
| mood | DELETE /mood/status | rest | flow | ../flows/mood-status-visibility.md |
| movie-match | POST /movie-match/sessions | rest | flow | ../flows/movie-match-friend-session.md |
| movie-match | PATCH /movie-match/sessions/:id/state | rest | flow | ../flows/movie-match-friend-session.md |
| movie-match | POST /movie-match/sessions/:id/end | rest | flow | ../flows/movie-match-friend-session.md |
| movie-match | GET /movie-match/sessions/:id | rest | flow | ../flows/movie-match-friend-session.md |
| movie-match | POST /movie-match/anon/queue | rest | flow | ../flows/movie-match-anonymous.md |
| movie-match | GET /movie-match/anon/current | rest | flow | ../flows/movie-match-anon-chat-finish-reactions.md |
| movie-match | DELETE /movie-match/anon/current | rest | flow | ../flows/movie-match-anon-chat-finish-reactions.md |
| movie-match | PATCH /movie-match/anon/sessions/:id/state | rest | flow | ../flows/movie-match-anonymous.md |
| movie-match | POST /movie-match/anon/sessions/:id/finish | rest | flow | ../flows/movie-match-anon-chat-finish-reactions.md |
| movie-match | POST /movie-match/anon/sessions/:id/rating | rest | flow | ../flows/movie-match-anonymous.md |
| movie-match | POST /movie-match/anon/sessions/:id/messages | rest | flow | ../flows/movie-match-anon-chat-finish-reactions.md |
| movie-match | GET /movie-match/anon/sessions/:id/messages | rest | flow | ../flows/movie-match-anon-chat-finish-reactions.md |
| movie-match | POST /movie-match/anon/sessions/:id/reactions | rest | flow | ../flows/movie-match-anon-chat-finish-reactions.md |
| mini-game | POST /mini-game/sessions | rest | flow | ../flows/mini-game-session.md |
| mini-game | POST /mini-game/sessions/:id/moves | rest | flow | ../flows/mini-game-session.md |
| mini-game | POST /mini-game/sessions/:id/cancel | rest | local |  |
| mini-game | GET /mini-game/sessions/:id | rest | local |  |
| friend | GET /profiles/:profileUserId/actions | rest | flow | ../flows/friend-profile-follow-and-actions.md |
| friend | POST /profiles/:profileUserId/follow | rest | flow | ../flows/friend-profile-follow-and-actions.md |
| friend | DELETE /profiles/:profileUserId/follow | rest | flow | ../flows/friend-profile-follow-and-actions.md |
| friend | POST /profiles/:profileUserId/conversation | rest | flow | ../flows/friend-open-conversation-from-profile.md |
| friend | GET /friends | rest | flow | ../flows/friend-list-and-conversation-reads.md |
| friend | GET /friends/:friendUserId/conversation | rest | flow | ../flows/friend-open-conversation-from-profile.md |
| friend | GET /conversations/:id/messages | rest | flow | ../flows/friend-list-and-conversation-reads.md |
| friend | GET /conversations/:id/streak | rest | flow | ../flows/friend-list-and-conversation-reads.md |
| friend | POST /conversations/:id/read | rest | flow | ../flows/friend-list-and-conversation-reads.md |
| friend | POST /conversations/:id/mute | rest | flow | ../flows/friend-list-and-conversation-reads.md |
| friend | POST /conversations/:id/messages | rest | flow | ../flows/message-push-notification.md |
| friend | StreakWarningJob.onApplicationBootstrap | job | flow | ../flows/friend-streak-warning-job.md |
| notification | GET /notifications | rest | local |  |
| notification | GET /notifications/unread-count | rest | local |  |
| notification | POST /notifications/web-push/subscription | rest | flow | ../flows/message-push-notification.md |
| notification | DELETE /notifications/web-push/subscription | rest | local |  |
| notification | POST /notifications/:notificationId/read | rest | local |  |
| discovery | GET /discovery/browse | rest | flow | ../flows/discovery-browse-and-nearby.md |
| discovery | PUT /discovery/nearby/location | rest | flow | ../flows/discovery-browse-and-nearby.md |
| discovery | PUT /discovery/nearby/visible | rest | flow | ../flows/discovery-browse-and-nearby.md |
| discovery | GET /discovery/nearby | rest | flow | ../flows/discovery-browse-and-nearby.md |
| avatar | GET /avatar/catalog | rest | local |  |
| avatar | GET /avatar/me/items | rest | local |  |
| avatar | POST /avatar/items/:assetId/claim | rest | flow | ../flows/avatar-buy-item.md |
| avatar | POST /avatar/items/:assetId/buy | rest | flow | ../flows/avatar-buy-item.md |
| avatar | PUT /avatar/me/equip | rest | flow | ../flows/avatar-buy-item.md |
| avatar | GET /avatar/me | rest | local |  |
| avatar | GET /avatar/users/:userId | rest | local |  |
| media | POST /media/images/upload-intent | rest | flow | ../flows/feed-post-with-image.md |
| feed | POST /stories | rest | flow | ../flows/story-create-with-image.md |
| feed | GET /stories/ring | rest | flow | ../flows/story-ring-view-and-viewers.md |
| feed | GET /stories/:storyId | rest | flow | ../flows/story-ring-view-and-viewers.md |
| feed | GET /stories/:storyId/viewers | rest | flow | ../flows/story-ring-view-and-viewers.md |
| feed | POST /stories/:storyId/reply | rest | flow | ../flows/story-reply-to-dm.md |
| feed | POST /feed/posts | rest | flow | ../flows/feed-post-with-image.md |
| feed | GET /feed/posts | rest | flow | ../flows/feed-read-posts.md |
| feed | GET /feed/users/:userId/posts | rest | flow | ../flows/feed-read-posts.md |
| feed | GET /feed/posts/:postId | rest | flow | ../flows/feed-read-posts.md |
| feed | DELETE /feed/posts/:postId | rest | flow | ../flows/feed-delete-post.md |
| feed | POST /feed/posts/:postId/comments | rest | flow | ../flows/feed-comment-with-notification.md |
| feed | GET /feed/posts/:postId/comments | rest | flow | ../flows/feed-comment-with-notification.md |
| feed | DELETE /feed/comments/:commentId | rest | flow | ../flows/feed-delete-post.md |
| feed | POST /feed/posts/:postId/reactions | rest | flow | ../flows/feed-reaction-with-notification.md |
| feed | DELETE /feed/posts/:postId/reactions | rest | flow | ../flows/feed-reaction-with-notification.md |
| feed | GET /feed/posts/:postId/reactions | rest | flow | ../flows/feed-reaction-with-notification.md |
| feed | StorySweeperService.onApplicationBootstrap | job | flow | ../flows/story-create-with-image.md |
| short-video | POST /videos/upload-intent | rest | flow | ../flows/short-video-publish-moderation.md |
| short-video | POST /videos/:id/finalize | rest | flow | ../flows/short-video-publish-moderation.md |
| short-video | GET /videos | rest | flow | ../flows/short-video-browse-and-detail.md |
| short-video | GET /videos/:id | rest | flow | ../flows/short-video-browse-and-detail.md |
| short-video | POST /videos/:id/report | rest | flow | ../flows/short-video-report.md |
| short-video | POST /videos/:id/views | rest | flow | ../flows/short-video-browse-and-detail.md |
| short-video | POST /videos/:id/reactions | rest | flow | ../flows/short-video-browse-and-detail.md |
| short-video | DELETE /videos/:id/reactions | rest | flow | ../flows/short-video-browse-and-detail.md |
| short-video | POST /videos/:id/comments | rest | flow | ../flows/short-video-comments.md |
| short-video | GET /videos/:id/comments | rest | flow | ../flows/short-video-comments.md |
| short-video | DELETE /videos/:id/comments/:commentId | rest | flow | ../flows/short-video-comments.md |
| short-video | VideoRankingService.onApplicationBootstrap | job | flow | ../flows/short-video-ranking-and-sweeper-jobs.md |
| short-video | VideoSweeperService.onApplicationBootstrap | job | flow | ../flows/short-video-ranking-and-sweeper-jobs.md |
| party-room | POST /party/rooms | rest | flow | ../flows/party-room-lifecycle.md |
| party-room | GET /party/rooms | rest | flow | ../flows/party-room-detail-and-comments.md |
| party-room | GET /party/rooms/:id | rest | flow | ../flows/party-room-detail-and-comments.md |
| party-room | GET /party/rooms/:id/comments | rest | flow | ../flows/party-room-detail-and-comments.md |
| party-room | POST /party/rooms/:id/comments | rest | flow | ../flows/party-room-detail-and-comments.md |
| party-room | POST /party/rooms/:id/join | rest | flow | ../flows/party-room-lifecycle.md |
| party-room | POST /party/rooms/:id/leave | rest | flow | ../flows/party-room-leave-and-role-change.md |
| party-room | POST /party/rooms/:id/members/:userId/role | rest | flow | ../flows/party-room-leave-and-role-change.md |
| party-room | POST /party/rooms/:id/members/:userId/speaker-invite | rest | flow | ../flows/party-room-speaker-invite.md |
| party-room | POST /party/rooms/:id/speaker-invite/accept | rest | flow | ../flows/party-room-speaker-invite.md |
| party-room | POST /party/rooms/:id/speaker-invite/decline | rest | flow | ../flows/party-room-speaker-invite.md |
| party-room | POST /party/webhooks/livekit | webhook | flow | ../flows/party-room-lifecycle.md |
| party-room | PartyRoomSweeperService.onApplicationBootstrap | job | flow | ../flows/party-room-lifecycle.md |
| safety | POST /safety/reports | rest | flow | ../flows/safety-report-and-trust-score.md |
| safety | POST /safety/blocks/:targetUserId | rest | flow | ../flows/safety-block-and-unblock.md |
| safety | DELETE /safety/blocks/:targetUserId | rest | flow | ../flows/safety-block-and-unblock.md |
| safety | GET /safety/blocks/:targetUserId | rest | flow | ../flows/safety-block-and-unblock.md |
| signaling-gateway | GET /health | rest | local |  |
| signaling-gateway | GET /health/live | rest | local |  |
| signaling-gateway | GET /health/ready | rest | local |  |
| friend | GET /friends/connections | rest | flow | ../flows/friend-follow-lists-and-connections.md |
| friend | GET /profiles/:profileUserId/follow-counts | rest | flow | ../flows/friend-follow-lists-and-connections.md |
| friend | GET /profiles/:profileUserId/followers | rest | flow | ../flows/friend-follow-lists-and-connections.md |
| friend | GET /profiles/:profileUserId/following | rest | flow | ../flows/friend-follow-lists-and-connections.md |
