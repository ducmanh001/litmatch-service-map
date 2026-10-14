# Chặn và bỏ chặn người dùng

id: flow:safety-block-and-unblock
group: Phòng party & an toàn
services: module:safety, module:user

Người dùng chặn, bỏ chặn hoặc kiểm tra trạng thái chặn một người khác. Safety sở hữu log Block append-only nhưng phải hỏi User để chắc người bị chặn tồn tại trước khi ghi.

Ba endpoint /safety/blocks/:targetUserId đều đọc dòng Block mới nhất trong PostgreSQL; chỉ block gọi thêm User, và chỉ ghi dòng mới khi trạng thái thật sự đổi.

## Participants

- web (client): Web client
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> safety [call/http]: POST /safety/blocks/:targetUserId (chặn)
2. safety -> user [call/di-call]: Kiểm tra người bị chặn tồn tại (getByIdOrThrow) (Tự chặn chính mình bị từ chối trước bước này bằng SELF_BLOCK_FORBIDDEN (400).) {apps/core-api/src/modules/safety/safety.service.ts:199}
3. safety -> postgres [call/shared-db]: Đọc dòng Block mới nhất của cặp (latestAction) {apps/core-api/src/modules/safety/safety.service.ts:376}
4. safety -> postgres [call/shared-db]: Ghi dòng Block action=Blocked (bỏ qua nếu đã Blocked) (Idempotent: nếu dòng mới nhất đã là Blocked thì return, không ghi dòng trùng.) {apps/core-api/src/modules/safety/safety.service.ts:202}
5. safety -> web [return/http]: 204 No Content
6. web -> safety [call/http]: DELETE /safety/blocks/:targetUserId (bỏ chặn)
7. safety -> postgres [call/shared-db]: Đọc dòng Block mới nhất của cặp (latestAction) {apps/core-api/src/modules/safety/safety.service.ts:376}
8. safety -> postgres [call/shared-db]: Ghi dòng Block action=Unblocked (bỏ qua nếu chưa Blocked) (Idempotent: nếu dòng mới nhất không phải Blocked thì return, không ghi gì.) {apps/core-api/src/modules/safety/safety.service.ts:215}
9. safety -> web [return/http]: 204 No Content
10. web -> safety [call/http]: Đọc dòng Block mới nhất để suy ra trường blocked
11. safety -> postgres [call/shared-db]: Đọc dòng Block mới nhất để suy ra isBlocked {apps/core-api/src/modules/safety/safety.service.ts:376}
12. safety -> web [return/http]: BlockStatusDto: đang chặn hay không

## Invariants

- Block là log append-only: trạng thái hiện tại là action của dòng mới nhất theo createdAt DESC (latestAction), nên block/unblock chỉ thêm dòng, không sửa hay xóa dòng cũ.
- Safety phải hỏi UserService.getByIdOrThrow trước khi ghi để không tạo Block cho user không tồn tại; bước này nằm trước đọc latestAction và ghi.
- Block và unblock idempotent: đọc latestAction trước khi ghi giúp lặp lại cùng thao tác không sinh dòng thừa. Suy luận: không có transaction hay khoá nên hai request đồng thời vẫn có thể ghi dòng trùng.

## Failure modes

- when blockerUserId trùng targetUserId: SafetyService.block ném DomainException SELF_BLOCK_FORBIDDEN (400), không gọi User và không ghi Block.
- when targetUserId không tồn tại: UserService.getByIdOrThrow ném lỗi, request thất bại và không có dòng Block nào được ghi.
- when targetUserId không phải UUID hợp lệ: ParseUUIDPipe trên controller từ chối request trước khi vào SafetyService.

## Endpoints covered

- POST /safety/blocks/:targetUserId [module:safety] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/safety.md
- DELETE /safety/blocks/:targetUserId [module:safety] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/safety.md
- GET /safety/blocks/:targetUserId [module:safety] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/safety.md
