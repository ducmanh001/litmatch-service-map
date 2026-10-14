# Báo cáo người dùng và trừ điểm tin cậy

id: flow:safety-report-and-trust-score
group: Phòng party & an toàn
services: module:safety, module:user

Khi một người dùng báo cáo người khác, safety phải ghi Report append-only và trừ trust score của người bị báo cáo atomic, có trần theo cặp và theo ngày; vì vậy safety phối hợp với user (sở hữu bảng users) và PostgreSQL trong cùng một transaction.

Luồng POST /safety/reports: kiểm tra đích, khoá dòng user, tính mức phạt, ghi Report và trừ trust score trong một transaction.

## Participants

- web (client): Web client
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> safety [call/http]: POST /safety/reports {targetUserId, reason, description} (Route có JWT guard toàn cục và @Throttle 10 request/60 phút mỗi tài khoản.) {apps/core-api/src/modules/safety/safety.controller.ts:37}
2. safety -> user [call/di-call]: Kiểm tra người bị báo cáo tồn tại (getByIdOrThrow) (Trước đó tự báo cáo chính mình bị từ chối với 400 SELF_REPORT_FORBIDDEN.) {apps/core-api/src/modules/safety/safety.service.ts:66}
3. safety -> postgres [call/shared-db]: Mở transaction, khoá pessimistic_write dòng User của người bị báo cáo (Khoá tuần tự hoá mọi lần phạt cùng một target, tránh vượt trần khi report đồng thời.) {apps/core-api/src/modules/safety/safety.service.ts:83, apps/core-api/src/modules/safety/safety.service.ts:89}
4. safety -> postgres [call/shared-db]: Đếm report đã phạt cùng cặp trong cooldown và tổng phạt 24h (penalty = 0 nếu cặp đã bị phạt trong cooldown, ngược lại min(penaltyPerReport, remainingCap).) {apps/core-api/src/modules/safety/safety.service.ts:94, apps/core-api/src/modules/safety/safety.service.ts:106}
5. safety -> postgres [call/shared-db]: INSERT Report kèm trustPenaltyApplied (luôn ghi để giữ evidence) {apps/core-api/src/modules/safety/safety.service.ts:118}
6. safety -> user [call/di-call]: Nếu penalty > 0: adjustTrustScore(manager, target, -penalty, floor) (Truyền cùng manager nên chạy trong transaction của safety.) {apps/core-api/src/modules/safety/safety.service.ts:128}
7. user -> postgres [call/shared-db]: UPDATE users.trust_score = GREATEST(floor, trust_score + delta) {apps/core-api/src/modules/user/user.service.ts:222}
8. safety -> web [return/http]: Commit, trả 201 ReportDto

## Invariants

- Report và trừ trust score phải cùng một transaction để không có report được tính phạt mà điểm không giảm (hoặc ngược lại).
- Dòng User bị khoá pessimistic_write trước khi đọc tổng phạt trong ngày, nếu không hai report đồng thời cùng đọc một tổng và vượt trần.
- Điểm tin cậy chỉ bị đổi qua UserService.adjustTrustScore với manager của safety; tuy nhiên safety tự khoá dòng users bằng manager.getRepository(User).findOne (chỉ đọc có khoá, không ghi).

## Failure modes

- when Người báo cáo trùng người bị báo cáo: Ném DomainException SELF_REPORT_FORBIDDEN (400), chưa mở transaction nào.
- when Người bị báo cáo không tồn tại: getByIdOrThrow ném lỗi trước transaction, không ghi gì.
- when Lỗi giữa chừng trong transaction (ví dụ UPDATE trust score): Transaction rollback, Report cũng không được lưu.
- when Cặp đã bị phạt trong cooldown hoặc đã chạm trần ngày: Report vẫn được ghi với trustPenaltyApplied = 0 và không gọi adjustTrustScore.

## Endpoints covered

- POST /safety/reports [module:safety] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/safety.md
