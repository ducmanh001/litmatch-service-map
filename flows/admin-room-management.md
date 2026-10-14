# Admin xem và đóng phòng party

id: flow:admin-room-management
group: Vận hành (admin)
services: module:admin, module:party-room, module:media-server, module:signaling-gateway

Admin liệt kê phòng party đang live rồi đóng một phòng. Việc này chạm nhiều dịch vụ: AdminModule kiểm quyền và ghi audit, Party Room sở hữu dữ liệu phòng, LiveKit phải dọn phòng âm thanh, và gateway báo realtime cho thành viên đang ở trong phòng.

Admin đóng phòng: đóng phòng và ghi audit cùng một transaction, sau commit mới dọn LiveKit và đẩy sự kiện realtime tới thành viên.

## Participants

- admin-console (client): Admin console
- admin (service): Admin API
- party-room (service): Party Room
- postgres (infra): PostgreSQL
- media-server (service): LiveKit SFU
- redis (infra): Redis
- signaling-gateway (service): Signaling gateway
- web (client): Web client (thành viên)

## Steps

1. admin-console -> admin [call/http]: GET /admin/rooms: xin danh sách phòng kèm số member
2. admin -> party-room [call/di-call]: Gọi listRoomsWithMemberCounts(limit, cursor) (Endpoint yêu cầu quyền ManageRooms qua AdminPermissionGuard.) {apps/core-api/src/modules/admin/admin.service.ts:234}
3. party-room -> postgres [call/shared-db]: Đọc trang phòng và đếm member chưa rời (2 query) {apps/core-api/src/modules/party-room/party-room.service.ts:948}
4. admin -> admin-console [return/http]: Trả danh sách phòng và memberCount
5. admin-console -> admin [call/http]: POST /admin/rooms/:id/close: đóng phòng được chọn
6. admin -> party-room [call/di-call]: Gọi closeRoomById với lý do AdminClosed và hook audit {apps/core-api/src/modules/admin/admin.service.ts:214}
7. party-room -> postgres [call/shared-db]: Khóa row phòng, đặt status Closed, lưu closeReason (Phòng không tồn tại hoặc đã Closed thì trả closed=false, không làm gì thêm (idempotent).) {apps/core-api/src/modules/party-room/party-room.service.ts:1033}
8. party-room -> admin [call/di-call]: Gọi hook onTransition trong cùng transaction {apps/core-api/src/modules/party-room/party-room.service.ts:1034}
9. admin -> postgres [call/shared-db]: Ghi admin_audit_logs party_room.admin_closed bằng manager chung {apps/core-api/src/modules/admin/admin.service.ts:219}
10. party-room -> postgres [call/shared-db]: Đặt leftAt cho mọi member đang active {apps/core-api/src/modules/party-room/party-room.service.ts:1036}
11. party-room -> media-server [call/sdk]: Sau commit: deleteRoom phòng LiveKit party-<id> (Lỗi deleteRoom bị bắt và chỉ ghi log debug.) {apps/core-api/src/modules/party-room/party-room.service.ts:1046}
12. party-room -> redis [async/redis-pubsub]: Publish PartyRoomClosed cho từng member đã rời (Lỗi publish được bắt và chỉ ghi warn; client còn polling fallback.) {apps/core-api/src/common/realtime/redis-realtime-publisher.adapter.ts:11}
13. redis -> signaling-gateway [async/redis-pubsub]: Gateway PSUBSCRIBE realtime:user:* nhận envelope {apps/signaling-gateway/src/app/signaling.gateway.ts:346}
14. signaling-gateway -> web [async/socket]: Emit sự kiện đóng phòng tới socket room của user {apps/signaling-gateway/src/app/signaling.gateway.ts:332}
15. party-room -> admin [return/di-call]: Trả kết quả { closed }
16. admin -> admin-console [return/http]: Trả { closed } cho admin

## Invariants

- Audit log được ghi bằng cùng manager với việc đóng phòng, nên không có phòng bị đóng mà thiếu dấu vết admin, và nếu audit lỗi thì việc đóng phòng rollback.
- Dọn LiveKit và publish realtime chạy sau khi transaction commit, vì cuộc gọi nhà cung cấp không nằm trong tính nguyên tử của PostgreSQL và không được giữ khóa row phòng.
- Chỉ lời gọi thực hiện chuyển trạng thái (closed=true) mới dọn SFU và publish, nên gọi đóng lặp lại không bắn sự kiện đôi.

## Failure modes

- when Phòng không tồn tại hoặc đã Closed: closeRoomById trả closed=false, không audit, không dọn LiveKit, không publish.
- when LiveKit deleteRoom lỗi (ví dụ phòng đã tự đóng): Lỗi bị bắt và ghi log debug; phòng DB vẫn Closed, luồng tiếp tục publish.
- when Publish Redis lỗi: Lỗi được bắt và ghi warn; DB đã Closed, client dựa vào polling fallback.
- when Ghi audit lỗi trong hook: Transaction rollback, phòng không bị đóng.

## Endpoints covered

- POST /admin/rooms/:id/close [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
- GET /admin/rooms [module:admin] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/admin.md
