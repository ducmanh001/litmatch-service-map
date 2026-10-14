# Palm Match: vào hàng, lật và đánh giá

id: flow:palm-match-queue-and-rating
group: Ghép đôi & gọi
services: module:palm-match, module:safety, module:friend

Người dùng vào hàng đợi ghép cặp ẩn danh, lật bài rồi like/skip. Palm Match sở hữu hàng đợi và phiên nhưng không sở hữu block/report (Safety) hay quan hệ bạn bè (Friend), nên phải hỏi Safety.canPair lúc ghép và nhờ Friend tạo Friendship khi cả hai like.

Luồng Palm Match: xem bói, vào hàng (kiểm canPair), poll, lật, đánh giá (like hai chiều mở friendship) và huỷ.

## Participants

- web (client): Web client
- palm-match (service): Palm Match
- safety (service): Safety
- friend (service): Friend
- postgres (infra): PostgreSQL

## Steps

1. web -> palm-match [call/http]: GET /palm-match/reading: xem bói theo category
2. palm-match -> postgres [call/shared-db]: Đọc palm_reading_templates, chọn nội dung theo seed user+ngày {apps/core-api/src/modules/palm-match/palm-match.service.ts:86}
3. web -> palm-match [call/http]: POST /palm-match/queue: vào hàng đợi ghép cặp
4. palm-match -> postgres [call/shared-db]: Khoá advisory, hết hạn phiên cũ, dọn queue quá hạn, enqueue, chọn ứng viên {apps/core-api/src/modules/palm-match/palm-match.service.ts:329}
5. palm-match -> safety [call/di-call]: canPair(userId, candidate): kiểm lại block và báo cáo gần đây lúc ghép (Duyệt tối đa 20 ứng viên cũ nhất; ứng viên đang có phiên active thì bị xoá khỏi queue và bỏ qua. canPair chạy bằng repository của Safety, không dùng manager của transaction.) {apps/core-api/src/modules/palm-match/palm-match.service.ts:158}
6. palm-match -> postgres [call/shared-db]: Tạo session snapshot, xoá queue hai người, insert active participant {apps/core-api/src/modules/palm-match/palm-match.service.ts:179}
7. palm-match -> web [return/http]: Trả state Queued (chờ) hoặc session đã ghép
8. web -> palm-match [call/http]: GET /palm-match/current: poll/khôi phục state
9. palm-match -> postgres [call/shared-db]: Lazy-expire phiên, đọc active state hoặc queue, xoá entry quá hạn {apps/core-api/src/modules/palm-match/palm-match.service.ts:190}
10. web -> palm-match [call/http]: POST /palm-match/sessions/:id/flip: lật lá của mình
11. palm-match -> postgres [call/shared-db]: Khoá phiên, kiểm còn active, ghi lowFlippedAt/highFlippedAt {apps/core-api/src/modules/palm-match/palm-match.service.ts:229}
12. web -> palm-match [call/http]: POST /palm-match/sessions/:id/rating: like hoặc skip
13. palm-match -> postgres [call/shared-db]: Khoá phiên, kiểm cả hai đã lật, ghi rating của caller {apps/core-api/src/modules/palm-match/palm-match.service.ts:240}
14. palm-match -> friend [call/di-call]: Cả hai Like: ensureFriendship(manager, low, high, PalmMatch) {apps/core-api/src/modules/palm-match/palm-match.service.ts:286}
15. friend -> postgres [call/shared-db]: Insert Friendship và Conversation (orIgnore) bằng manager của Palm Match {apps/core-api/src/modules/friend/friend.service.ts:117}
16. palm-match -> postgres [call/shared-db]: Chốt outcome Matched/NotMatched và lưu session {apps/core-api/src/modules/palm-match/palm-match.service.ts:295}
17. palm-match -> web [return/http]: Trả state phiên sau đánh giá
18. web -> palm-match [call/http]: DELETE /palm-match/current: huỷ queue hoặc phiên
19. palm-match -> postgres [call/shared-db]: Khoá matcher, xoá queue, huỷ phiên active, xoá active participant {apps/core-api/src/modules/palm-match/palm-match.service.ts:305}
20. palm-match -> web [return/http]: Trả state Idle

## Invariants

- Advisory lock được lấy đầu transaction vào hàng nên việc chọn cặp bị serialize; canPair được gọi lại ngay lúc ghép thay vì tin state lúc enqueue, vì block/report có thể đổi giữa hai thời điểm.
- Friendship và Conversation được tạo bằng cùng EntityManager với việc lưu session, nên mutual-like và kết quả Matched cùng commit hoặc cùng rollback; Friend sở hữu cách tạo, Palm Match chỉ gọi.
- Mọi route REST đều cần JWT (JwtAuthGuard toàn cục); userId lấy từ JWT, body không nhận participant.

## Failure modes

- when Safety.canPair trả false cho mọi ứng viên: Không có đối tác, user vẫn ở hàng đợi và nhận state Queued.
- when ensureFriendship ném lỗi trong transaction đánh giá: Transaction rollback, rating và outcome không được lưu.
- when Một người rate trước khi cả hai đã lật bài: Service ném RATING_NOT_OPEN (409), không ghi rating.
- when Đổi rating sau khi đã chốt: Service ném RATING_CONFLICT (409).

## Endpoints covered

- GET /palm-match/reading [module:palm-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/palm-match.md
- POST /palm-match/queue [module:palm-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/palm-match.md
- GET /palm-match/current [module:palm-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/palm-match.md
- POST /palm-match/sessions/:id/flip [module:palm-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/palm-match.md
- POST /palm-match/sessions/:id/rating [module:palm-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/palm-match.md
- DELETE /palm-match/current [module:palm-match] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/palm-match.md
