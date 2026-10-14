# Phân quyền dùng RBAC, ABAC hay kết hợp?

id: qa:authz-model
category: authz
services: module:admin, module:matching, module:safety, module:friend, module:party-room

Kết hợp nhiều lớp và không có policy engine (không CASL, Casbin hay OPA trong package.json): RBAC cho vai trò (RolesGuard toàn cục + ma trận quyền admin lưu DB), ABAC theo thuộc tính (guest, bị ban, hạng VIP, chính sách ghép đôi), ReBAC theo quan hệ (bạn bè, block), kiểm tra chủ sở hữu từng tài nguyên và capability token cho media.

### RBAC

RolesGuard đăng ký toàn cục đọc @Roles; khu admin còn có AdminPermissionGuard và bảng admin_role_permissions nên đổi quyền của một vai trò không cần deploy lại.

services: admin; evidence: apps/core-api/src/app/app.module.ts:154, apps/core-api/src/modules/admin/admin.controller.ts:64

### ABAC

Quyết định dựa trên thuộc tính: tài khoản bị ban bị chặn ngay lúc thao tác, hạng VIP được tra khi dựng thông tin ghép đôi, và MATCH_INTERACTION_POLICY quyết định hai người có được ghép với nhau không.

services: matching, soul-match; evidence: apps/core-api/src/modules/matching/matching.service.ts:109, apps/core-api/src/modules/soul-match/soul-match.service.ts:138

### ReBAC

Cho phép hoặc chặn theo quan hệ giữa hai người: canPair khi ghép đôi, kiểm block khi nhắn hay xem hồ sơ, areFriends khi mở phòng hoặc chat.

services: matching, friend, safety; evidence: apps/core-api/src/modules/matching/jobs/matcher-worker.service.ts:277, apps/core-api/src/modules/friend/friend.service.ts:181

### Chủ sở hữu và capability

Mỗi tài nguyên kiểm chủ sở hữu (ví dụ ticket.userId so với người gọi) để chặn IDOR; quyền trong phòng media là capability token LiveKit do core-api cấp theo vai trò.

services: matching, party-room; evidence: apps/core-api/src/modules/matching/matching.service.ts:650, apps/core-api/src/modules/party-room/clients/livekit-party-room.adapter.ts:24

## Alternatives not chosen

- Policy engine (OPA, Casbin, CASL): Suy luận: số vai trò ít và quy tắc gắn chặt dữ liệu nghiệp vụ nằm sẵn trong từng service, nên thêm engine là thêm vận hành mà chưa có lợi rõ rệt.
- ABAC thuần cho cả khu admin: RBAC dễ kiểm toán hơn cho vài vai trò và ma trận quyền sửa được lúc chạy.

## Caveats

- Suy luận: chính sách nằm rải trong mã từng module nên không có một nơi để xem toàn bộ quy tắc; muốn kiểm toán phải đọc từng service.
