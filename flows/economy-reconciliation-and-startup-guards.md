# Đối soát sổ cái và guard khởi động của verifier

id: flow:economy-reconciliation-and-startup-guards
group: Tiền tệ & quà
services: module:economy

Khi core-api khởi động, economy đăng ký hai job đối soát chỉ đọc (fast và deep) để tự phát hiện lệch Nợ/Có, và các verifier dev từ chối khởi động ở production. Cần Postgres (khóa advisory + ledger) và hệ metrics để cảnh báo lệch.

Lúc bootstrap economy đăng ký job đối soát đọc-only chạy dưới khóa advisory, đẩy metric lệch, rồi các verifier dev tự chặn ở production.

## Participants

- economy (service): Economy
- postgres (infra): PostgreSQL
- external:metrics (external): Metrics / Grafana

## Steps

1. economy -> economy [call/config]: Bootstrap: nếu ECONOMY_RECONCILIATION_ENABLED bật thì đăng ký interval fast (Một cờ tắt cả hai tier; nếu false thì onApplicationBootstrap return sớm.) {apps/core-api/src/modules/economy/jobs/reconciliation.service.ts:85}
2. economy -> postgres [call/shared-db]: Mỗi tick: mở transaction, giành advisory lock để chỉ một replica chạy (ManagedInterval.runClusterSingleton; replica không giành được lock thì rollback và hoãn retry.) {apps/core-api/src/common/scheduling/managed-interval.ts:128}
3. economy -> postgres [call/shared-db]: runFast: SUM Nợ/Có theo currency trên ledger_entries (Query chạy qua dataSource.query (connection khác với runner giữ advisory lock); lock vẫn giữ suốt lúc task chạy.) {apps/core-api/src/modules/economy/jobs/reconciliation.service.ts:131}
4. postgres -> economy [return/shared-db]: Trả các currency lệch (rỗng nếu cân)
5. economy -> postgres [call/shared-db]: Đếm iap_receipts đã credited nhưng thiếu transaction_id {apps/core-api/src/modules/economy/jobs/reconciliation.service.ts:141}
6. postgres -> economy [return/shared-db]: Trả số receipt mồ côi
7. economy -> external:metrics [async/sdk]: Ghi metric mismatch (invariant, orphan_receipt) và kết quả run fast (Lệch chỉ log error và đẩy metric để alert; job không tự sửa dữ liệu.) {apps/core-api/src/modules/economy/jobs/reconciliation.service.ts:158}
8. economy -> postgres [call/shared-db]: Tick deep: giành advisory lock riêng cho job deep {apps/core-api/src/common/scheduling/managed-interval.ts:128}
9. economy -> postgres [call/shared-db]: runDeep: lấy mẫu 100 ví mới cập nhật, so snapshot với số dư derive từ ledger {apps/core-api/src/modules/economy/jobs/reconciliation.service.ts:185}
10. postgres -> economy [return/shared-db]: Trả danh sách ví lệch snapshot so với derived
11. economy -> external:metrics [async/sdk]: Ghi metric walletMismatch và kết quả run deep {apps/core-api/src/modules/economy/economy.metrics.ts:101}
12. economy -> economy [call/config]: DevIapVerifier: NODE_ENV=production + verifier=dev thì throw, app không lên {apps/core-api/src/modules/economy/ports/iap-verifier.ts:67}
13. economy -> economy [call/config]: DevAppleNotificationVerifier: cùng guard với ECONOMY_APPLE_WEBHOOK_VERIFIER=dev {apps/core-api/src/modules/economy/ports/notification-verifier.ts:73}
14. economy -> economy [call/config]: DevGoogleRtdnVerifier: chặn ECONOMY_GOOGLE_RTDN_VERIFIER=dev ở production {apps/core-api/src/modules/economy/ports/notification-verifier.ts:202}

## Invariants

- Job đối soát chỉ SELECT, không tự sửa; sửa lệch thật phải qua reversal entry ở LedgerService nên đối soát tách khỏi write path.
- Advisory lock theo jobName (transaction-scoped) đảm bảo chỉ một replica chạy mỗi tier tại một thời điểm, tránh query đắt chạy trùng.
- Verifier dev không verify chữ ký nên phải bị chặn lúc bootstrap ở production, trước khi nhận bất kỳ webhook hay receipt nào.

## Failure modes

- when Query đối soát ném lỗi: scheduledRun nuốt lỗi, ghi recordReconciliationRun(tier,false) và log error để alert thấy job chết; interval vẫn sống.
- when Replica khác đang giữ advisory lock: Replica này rollback, đặt clusterRetryAfter và bỏ qua tick.
- when Verifier dev được cấu hình ở production: onApplicationBootstrap throw Error, tiến trình khởi động thất bại.

## Endpoints covered

- ReconciliationService.onApplicationBootstrap [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- DevIapVerifier.onApplicationBootstrap [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- DevAppleNotificationVerifier.onApplicationBootstrap [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
- DevGoogleRtdnVerifier.onApplicationBootstrap [module:economy] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/economy.md
