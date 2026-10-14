# Tìm người theo số điện thoại

id: flow:auth-search-by-phone
group: Danh tính
services: module:auth, module:user

Người dùng đã đăng nhập tìm hồ sơ người khác bằng số điện thoại. Module auth giữ danh tính điện thoại còn module user giữ quyền riêng tư và hồ sơ, nên hai service phải phối hợp để chỉ lộ hồ sơ khi chủ hồ sơ cho phép.

Auth tra danh tính theo số điện thoại, hỏi user về quyền riêng tư và trạng thái rồi trả hồ sơ công khai hoặc null.

## Participants

- web (client): Web client
- auth (service): Auth
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> auth [call/http]: GET /auth/search-by-phone?phone=... (JWT, tối đa 10 lần/phút) (Route có @Throttle 10 request mỗi phút; mọi route REST cần JWT trừ khi @Public.) {apps/core-api/src/modules/auth/controllers/phone-search.controller.ts:21}
2. auth -> postgres [call/shared-db]: Tìm AuthIdentity theo provider=Phone và providerUid=phone (Không có identity thì trả null ngay.) {apps/core-api/src/modules/auth/services/phone-search.service.ts:24}
3. auth -> user [call/di-call]: Hỏi isSearchableByPhone(identity.userId) (Nếu tắt quyền tìm theo số điện thoại thì trả null, giống trường hợp không có identity để không tạo oracle dò số.) {apps/core-api/src/modules/auth/services/phone-search.service.ts:28}
4. user -> postgres [call/shared-db]: Đọc user_privacy_settings, lấy cờ searchableByPhone (Chưa có dòng cài đặt thì dùng mặc định searchableByPhone=false, nên user chưa bật thì không tìm thấy.) {apps/core-api/src/modules/user/services/privacy-settings.service.ts:59}
5. user -> auth [return/di-call]: Trả true/false cho cờ searchableByPhone
6. auth -> user [call/di-call]: getByIdOrThrow(identity.userId) để lấy hồ sơ user {apps/core-api/src/modules/auth/services/phone-search.service.ts:32}
7. user -> auth [return/di-call]: Trả entity user (kèm status)
8. auth -> web [return/http]: Trả PublicProfileDto nếu status Active, ngược lại trả null (Nhánh null gồm: không có identity, tắt quyền tìm kiếm, user không Active.)

## Invariants

- Danh tính điện thoại thuộc auth còn cờ searchableByPhone và hồ sơ thuộc user, nên auth phải gọi user qua PrivacySettingsService và UserService thay vì tự đọc bảng của user.
- Phải kiểm quyền riêng tư trước khi đọc hồ sơ, và các nhánh không có identity, tắt tìm kiếm đều trả null như nhau để người gọi không dò được số nào đã đăng ký.

## Failure modes

- when Số điện thoại không có AuthIdentity, hoặc chủ hồ sơ tắt/chưa bật searchableByPhone (mặc định false): API trả null, không phân biệt hai trường hợp
- when User không ở trạng thái Active: API trả null thay vì hồ sơ
- when Gọi quá 10 lần mỗi phút: Bị throttler chặn theo cấu hình @Throttle của route

## Endpoints covered

- GET /auth/search-by-phone [module:auth] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/auth.md
