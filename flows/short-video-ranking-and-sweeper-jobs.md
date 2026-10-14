# Job tính điểm xếp hạng và dọn video lỗi

id: flow:short-video-ranking-and-sweeper-jobs
group: Xã hội & nội dung
services: module:short-video

Hai worker nền của short-video chạy theo setInterval: một job tính lại rank_score cho video published, một job chuyển video uploading quá hạn sang failed rồi xóa object trên storage. Cả hai cần PostgreSQL (khóa advisory để chỉ một replica chạy, và dữ liệu videos) cùng storage ngoài.

Hai job nền singleton của short-video đọc/ghi bảng videos trên PostgreSQL và dọn object lỗi trên storage ngoài.

## Participants

- short-video (service): Short-video
- postgres (infra): PostgreSQL
- external:video-storage (external): Video storage

## Steps

1. short-video -> short-video [call/config]: Bootstrap: đăng ký setInterval cho job ranking theo VIDEO_RANKING_JOB_INTERVAL_MS {apps/core-api/src/modules/short-video/jobs/video-ranking.service.ts:41}
2. short-video -> postgres [call/shared-db]: Mỗi tick: thử lấy advisory lock để chỉ một replica chạy job (Khóa là pg_try_advisory_xact_lock trong một transaction riêng của QueryRunner; replica không lấy được khóa thì rollback và chờ followerRetryMs.) {apps/core-api/src/common/scheduling/managed-interval.ts:128}
3. short-video -> postgres [call/shared-db]: UPDATE rank_score cho tối đa 1000 video published theo engagement và time-decay {apps/core-api/src/modules/short-video/jobs/video-ranking.service.ts:75}
4. postgres -> short-video [return/shared-db]: Trả số dòng đã cập nhật của tick ranking
5. short-video -> short-video [call/config]: Bootstrap: đăng ký setInterval cho job sweeper theo VIDEO_SWEEPER_INTERVAL_MS {apps/core-api/src/modules/short-video/jobs/video-sweeper.service.ts:45}
6. short-video -> postgres [call/shared-db]: Sau khi lấy khóa: chuyển video uploading quá hạn sang failed, lấy id, storage_key {apps/core-api/src/modules/short-video/jobs/video-sweeper.service.ts:69}
7. short-video -> postgres [call/shared-db]: Đọc thêm tối đa 200 video failed cũ để dọn lại object {apps/core-api/src/modules/short-video/jobs/video-sweeper.service.ts:87}
8. short-video -> external:video-storage [call/sdk]: VideoStoragePort.delete(storage_key) cho từng video cần dọn (Lỗi từng video bị bắt và chỉ log cảnh báo, video đó được thử lại ở tick sau.) {apps/core-api/src/modules/short-video/jobs/video-sweeper.service.ts:101}
9. short-video -> short-video [return/di-call]: Kết thúc tick sweeper, trả số video vừa chuyển sang failed

## Invariants

- Hai job chạy ngầm trong mọi replica core-api nhưng khóa advisory trên PostgreSQL (theo tên job) bảo đảm chỉ một replica thực hiện mỗi tick, tránh tính điểm hoặc dọn trùng.
- Sweeper phải ghi status failed vào PostgreSQL trước khi xóa object trên storage, để video lỗi vẫn được nhặt lại ở tick sau nếu việc xóa thất bại.

## Failure modes

- when VideoStoragePort.delete ném lỗi cho một video: Lỗi bị bắt và ghi log warn; video vẫn ở trạng thái failed nên được chọn lại ở tick sau.
- when Tick trước còn chạy hoặc replica khác giữ khóa advisory: Tick còn chạy trong cùng process thì bị bỏ qua; replica không lấy được khóa thì rollback và chờ followerRetryMs. Process idle cũng bỏ qua tick (skipWhenIdle).
- when Truy vấn PostgreSQL lỗi trong tick: Transaction giữ khóa được rollback, lỗi được log và replica này hoãn thử lại một khoảng retry.

## Endpoints covered

- VideoRankingService.onApplicationBootstrap [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
- VideoSweeperService.onApplicationBootstrap [module:short-video] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/short-video.md
