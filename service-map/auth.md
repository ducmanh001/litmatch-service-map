# Service map: Auth

id: service-map:auth
module: module:auth
deployable: core-api
confidence: high

Đăng nhập, phiên refresh và nâng cấp guest

Sở hữu danh tính đăng nhập (auth_identities), OTP điện thoại, refresh-token family và việc ký access token JWT / guest device token. Không sở hữu hồ sơ user (module user), không gửi SMS thật và không tự verify token ở các route khác (JwtAuthGuard global làm việc đó).

Owns: bảng auth_identities (provider, providerUid -> userId, unique); bảng phone_otps (codeHash HMAC, attemptCount, consumedAt); bảng refresh_tokens (tokenHash SHA-256, familyId, rotatedAt, revokedAt); cookie httpOnly refresh_token + csrf_token, guest device token

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | POST | /api/v1/auth/guest | public | Đăng nhập khách bằng deviceId: find-or-create user guest + identity trong một transaction, trả access token, set cookie refresh và cấp guestDeviceToken. Public, throttle 5/phút/IP. |
| e2 | rest | POST | /api/v1/auth/otp/request | public | Tạo OTP 6 số cho số điện thoại (rate limit theo số trong DB, vô hiệu mã cũ) và trả mã trong response vì không gửi SMS. Public, throttle 5/phút/IP; 403 nếu AUTH_PHONE_OTP_ENABLED tắt. |
| e3 | rest | POST | /api/v1/auth/otp/verify | public | Xác minh OTP (đếm lần thử và consume bằng UPDATE có điều kiện) rồi đăng nhập hoặc tạo user mới theo identity phone. Public, throttle 10/phút/IP. |
| e4 | rest | POST | /api/v1/auth/social | public | Đăng nhập Google/Apple (ID token JWKS) hoặc Facebook (debug_token) sau khi server tự verify, rồi find-or-create user theo (provider, uid). Public, throttle 10/phút/IP. |
| e5 | rest | POST | /api/v1/auth/refresh | refresh-cookie | Xoay refresh token từ cookie httpOnly (rotation, phát hiện reuse thu hồi cả family), kiểm lại user chưa bị ban rồi cấp access token mới. Cần CsrfGuard (cookie csrf_token khớp header x-csrf-token). |
| e6 | rest | POST | /api/v1/auth/logout | refresh-cookie | Thu hồi refresh token hiện tại (idempotent) và xoá cookie. Cần CsrfGuard; trả 204. |
| e7 | rest | POST | /api/v1/auth/upgrade/otp | jwt | Nâng cấp tài khoản guest đang đăng nhập bằng OTP điện thoại: gắn identity, xoá identity guest, đặt isGuest=false và thu hồi mọi refresh session trong một transaction có khoá hàng user. Cần JWT. |
| e8 | rest | POST | /api/v1/auth/upgrade/social | jwt | Nâng cấp guest bằng social ID token: cùng luồng linkIdentityAndUpgrade như OTP, 409 nếu identity đã thuộc tài khoản khác (không tự merge). Cần JWT. |
| e9 | rest | GET | /api/v1/auth/search-by-phone | jwt | Tìm hồ sơ công khai theo số điện thoại chỉ khi chủ hồ sơ bật cho phép; trả null cả khi không tồn tại lẫn khi tắt privacy để không tạo oracle. Cần JWT, throttle 10/phút. |

## Coordination edges (why this service talks to others)

### đọc/tạo user khi login (c1) with user [service, out, di-call]

- endpoints: e1, e3, e4, e5, e7, e8, e9
- what: UserService.getByIdOrThrow (nạp user để kiểm status, isGuest, role cho JWT) và createWithManager(manager, {nickname, isGuest}) để tạo user mới trong transaction của auth; PhoneSearchService còn gọi PrivacySettingsService.isSearchableByPhone.
- why: Hồ sơ user thuộc module user; auth chỉ giữ identity -> userId nên phải hỏi user để biết tài khoản có bị khoá (UserStatus) và role/isGuest nào đưa vào access token. Tạo user qua createWithManager với manager của auth để user + auth_identities commit cùng một transaction.
- if missing: Auth phải tự ghi bảng users, hoặc user mới tạo mà không có identity (hoặc ngược lại), và user đã bị ban vẫn refresh được phiên mới.
- evidence: apps/core-api/src/modules/auth/auth.service.ts:128, apps/core-api/src/modules/auth/auth.service.ts:162, apps/core-api/src/modules/auth/auth.service.ts:170
- review: sound

### khoá và flip isGuest (c2) with user [service, out, shared-db]

- endpoints: e7, e8
- what: Trong linkIdentityAndUpgrade: manager.findOne(User, lock pessimistic_write) rồi manager.update/save User.isGuest = false cùng transaction với insert/delete AuthIdentity và revokeForUser.
- why: Nâng cấp guest phải atomic: gắn identity mới, xoá credential guest, đổi cờ isGuest và thu hồi refresh session cũ hoặc không gì cả. Khoá hàng user chặn hai request upgrade song song; nhánh unique-violation dùng update có điều kiện isGuest=true để chỉ một bên thu hồi session. Cần chạm trực tiếp entity User vì UserService không có API nhận manager để đổi isGuest (chỉ có createWithManager, banUser/unbanUser, adjustTrustScore, setRoleWithManager). Matching khoá cùng hàng users trong GuestMatchQuotaService.authorize (guest-match-quota.service.ts:45-56) nên upgrade và join queue được tuần tự, và quyết định quota guest đọc isGuest từ DB chứ không từ JWT cũ.
- if missing: Có thể commit nửa chừng: guest vẫn giữ refresh session cũ thành quyền account thật, hoặc hai identity cùng gắn cho một guest.
- evidence: apps/core-api/src/modules/auth/auth.service.ts:225, apps/core-api/src/modules/auth/auth.service.ts:286
- review: sound

### identity, OTP, refresh (c3) with postgres [infra, out, shared-db]

- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9
- what: Đọc/ghi auth_identities (unique provider+providerUid), phone_otps (codeHash, attemptCount, consumedAt) và refresh_tokens (tokenHash, familyId, rotatedAt, revokedAt).
- why: Các bất biến an toàn dựa vào ràng buộc và UPDATE có điều kiện của Postgres: unique identity là chốt cuối khi hai login đầu tiên chạy song song, attempt_count < max và consumed_at IS NULL chặn brute-force/dùng lại OTP, rotated_at IS NULL + khoá hàng làm rotation refresh an toàn dưới race.
- if missing: Mất phát hiện reuse refresh token, OTP dùng lại hoặc brute-force được, và race đăng nhập lần đầu tạo hai user cho cùng một thiết bị/số.
- evidence: apps/core-api/src/modules/auth/entities/auth-identity.entity.ts:22, apps/core-api/src/modules/auth/entities/phone-otp.entity.ts:10, apps/core-api/src/modules/auth/entities/refresh-token.entity.ts:14
- review: sound

### verifyForUser device token (c4) with matching [service, in, di-call]

- endpoints: e1
- what: MatchingService (GuestMatchQuotaService) gọi GuestDeviceTokenService.verifyForUser(token, userId) để đổi header X-Guest-Device-Token thành deviceKey đã HMAC.
- why: GuestMatchQuotaService dựng 3 khoá băm user/device(deviceKey)/network(IP chuẩn hoá) trong guest_match_quotas (guest-match-quota.service.ts:71-75), nên guest tạo tài khoản mới trên cùng thiết bị hoặc mạng vẫn dính quota. Auth ký token HS256 (aud litmatch-matching, secret AUTH_GUEST_DEVICE_TOKEN_SECRET); verifyForUser kiểm purpose và sub == userId nên token tài khoản khác bị từ chối.
- if missing: Client tự khai deviceKey tuỳ ý ở mỗi request hoặc dùng token của tài khoản khác để né khoá device; matching phải lưu deviceId thô hoặc tin client. Lưu ý deviceId lúc login vẫn do client khai nên chặn thực sự còn dựa vào khoá network (IP).
- evidence: apps/core-api/src/modules/matching/services/guest-match-quota.service.ts:67
- review: corrected — Token chỉ ràng buộc sub và deviceKey lúc login, không chứng minh 'thiết bị khác' bị từ chối; deviceId do client khai nên chặn thực sự còn nhờ khoá network.

### JWT_SECRET dùng chung (c50) with signaling-gateway [service, out, config]

- endpoints: e1, e3, e4, e5, e7, e8
- what: core-api ký access token (payload sub, isGuest, role, jti) bằng JWT_SECRET; signaling-gateway cấu hình JwtModule với cùng JWT_SECRET và verifyAsync token ở handshake.auth.token. [10/10] Từ commit 80106a1/54c66ab có xoay khoá không đăng xuất mọi người: core-api và gateway ký bằng JWT_SECRET, chỉ khi CHỮ KÝ không khớp mới thử JWT_SECRET_PREVIOUS (token hết hạn hay sai claim không được thử lại); quy trình 3 bước vì rollout cuốn chiếu làm pod cũ từ chối token do pod mới ký; env validation từ chối JWT_SECRET_PREVIOUS trùng AUTH_GUEST_DEVICE_TOKEN_SECRET.
- why: Gateway không có DB và không gọi core-api để hỏi, nên cách duy nhất để biết socket thuộc userId nào là tự verify chữ ký token do auth ký. Hai deployable bắt buộc cấu hình cùng JWT_SECRET (>=32 ký tự); gateway chỉ verify, không bao giờ ký.
- if missing: Lệch JWT_SECRET: mọi handshake bị Unauthorized, realtime chết im lặng dù REST vẫn chạy. Nếu gateway phải gọi core-api để verify thì thêm phụ thuộc runtime và mất tính stateless.
- evidence: apps/core-api/src/modules/auth/auth.module.ts:32, apps/signaling-gateway/src/app/app.module.ts:31, apps/signaling-gateway/src/app/signaling.gateway.ts:214
- review: sound

### verify Google ID token (c51) with external:google [external, out, sdk]

- endpoints: e4, e8
- what: SocialVerifierService tải JWKS công khai (createRemoteJWKSet GOOGLE_JWKS_URL) và jwtVerify ID token với issuer Google và audience AUTH_GOOGLE_CLIENT_ID để lấy payload.sub.
- why: Server không được tin ID token client đưa lên (docs/10 §10.0.B); chỉ khoá công khai của Google mới chứng minh token do Google phát hành cho đúng app. Uid lấy từ sub làm providerUid của identity.
- if missing: Ai cũng giả được token Google và chiếm tài khoản bất kỳ, hoặc nâng cấp guest sang identity của người khác.
- evidence: apps/core-api/src/modules/auth/services/social-verifier.ts:31
- review: sound

### verify Apple ID token (c52) with external:apple [external, out, sdk]

- endpoints: e4, e8
- what: Cùng cơ chế OIDC: createRemoteJWKSet(APPLE_OIDC_JWKS_URL), jwtVerify với issuer APPLE_OIDC_ISSUER và audience AUTH_APPLE_CLIENT_ID.
- why: Đăng nhập Sign in with Apple trả JWT ký bởi Apple; server phải verify chữ ký, issuer, audience trước khi coi sub là danh tính, vì token đi qua client không đáng tin.
- if missing: Token Apple giả được chấp nhận, mở đường chiếm tài khoản; không có cách nào khác để chứng minh danh tính Apple.
- evidence: apps/core-api/src/modules/auth/services/social-verifier.ts:32
- review: sound

### Graph debug_token (c53) with external:facebook [external, out, http]

- endpoints: e4, e8
- what: fetch tới FACEBOOK_DEBUG_TOKEN_URL với input_token và access_token = appId|appSecret (timeout 5s), kiểm is_valid, app_id khớp AUTH_FACEBOOK_APP_ID và lấy user_id.
- why: Facebook cấp access token opaque (không phải JWT có JWKS) nên không tự verify cục bộ được; chỉ Graph debug_token xác nhận token còn hiệu lực và thuộc đúng app của mình, tránh token cấp cho app khác.
- if missing: Token Facebook của app khác hoặc đã hết hạn vẫn đăng nhập được; không có cách xác minh offline.
- evidence: apps/core-api/src/modules/auth/services/social-verifier.ts:111
- review: sound

### refresh cookie + CSRF (c54) with web [client, in, http]

- endpoints: e5, e6
- what: libs/api-client gọi POST /auth/refresh với credentials: 'include' và header x-csrf-token lấy từ csrf token của response trước; single-flight trong tab và Web Locks liên tab (navigator.locks, bỏ qua nếu browser thiếu API) để N request 401 chỉ xoay token một lần.
- why: Refresh token nằm trong cookie httpOnly (ADR 0007) nên JS không đọc được; vì cookie tự gắn vào request nên CsrfGuard bắt double-submit header. Rotation chỉ cho phép dùng mỗi token một lần: nếu client gửi hai refresh song song, request thua bị coi là reuse và thu hồi cả family, nên client phải single-flight.
- if missing: Client không gửi cookie/header CSRF thì refresh luôn 401; gửi refresh song song làm đăng xuất người dùng ngoài ý muốn.
- evidence: libs/api-client/src/lib/client.ts:131
- review: corrected — Khoá liên tab dùng Web Locks (navigator.locks) và bị bỏ qua khi browser thiếu API (Safari cũ); khi đó chỉ còn single-flight trong tab.

### guestDeviceToken cho client (c55) with web [client, out, http]

- endpoints: e1
- what: e1 trả guestDeviceToken trong AuthTokensDto; token-store của api-client giữ nó. client.ts:177-183 chỉ gắn header X-Guest-Device-Token cho đường dẫn matching tickets (isMatchingTicketsPath) và gắn lại khi retry sau refresh; sau refresh client giữ token cũ nếu response không trả token mới (client.ts:145-148).
- why: Token chỉ được cấp một lần lúc guest login và ràng buộc thiết bị; chỉ client mới giữ được nó giữa các request nên phải mang theo để matching kiểm quota guest theo thiết bị.
- if missing: Client không gửi header thì matching trả 401 cho guest (GuestMatchQuota bắt buộc X-Guest-Device-Token).
- evidence: libs/api-client/src/lib/token-store.ts:126
- review: sound

## Notes

- Không có docs/services cho auth; mô tả dựa trên code và ADR 0002/0003/0007 (chưa đối chiếu nội dung ADR từng dòng).
- POST /auth/otp/request KHÔNG gửi SMS: OtpService.requestOtp trả thẳng `code` trong response để client hiển thị toast/tự điền (comment trong code 'Không gửi SMS'). Đây là hành vi theo code, kể cả khi AUTH_PHONE_OTP_ENABLED bật; cần cổng cấu hình ở production.
- AuthController gắn @Public() ở cấp class nên /auth/refresh và /auth/logout bỏ qua JwtAuthGuard; thay vào đó dùng CsrfGuard (double-submit cookie + header x-csrf-token) và đọc refresh token từ cookie httpOnly.
- Throttler theo IP trên controller (5-10 req/phút, @Throttle theo route). ThrottlerModule.forRootAsync (app.module.ts:73-84) mặc định không cấu hình storage nên bộ đếm là in-memory theo từng instance (THROTTLE_STORAGE=memory); K8s đặt THROTTLE_STORAGE=redis nên dùng RedisThrottlerStorage (Lua, fallback bộ nhớ khi Redis lỗi), tracker mặc định theo IP, mặc định THROTTLE_TTL_SECONDS/THROTTLE_LIMIT cho route không có @Throttle; chỉ chia sẻ giữa replica khi bật redis. [10/10] Với redis bộ đếm khoá theo req.ip, nên K8s base đặt HTTP_TRUST_PROXY_HOPS=1 (nếu không mọi người dùng chung IP ingress) và core-api cảnh báo lúc boot khi redis + production + hops 0. Rate limit theo số điện thoại (AUTH_OTP_REQUESTS_PER_HOUR) nằm trong OtpService bằng bảng phone_otps, không dùng Redis.
- Ban chỉ có hiệu lực ở lần refresh kế tiếp: UserService.banUser (user.service.ts:232) chỉ đổi users.status và không gọi RefreshSessionPort.revokeForUser (chỉ linkIdentityAndUpgrade gọi nó). JwtAuthGuard không đọc DB nên access token đã cấp còn dùng được tối đa JWT_ACCESS_TTL_SECONDS (mặc định 900s, env.validation.ts:267); lúc refresh, AuthService.refresh thu hồi token vừa xoay rồi ném USER_BANNED (auth.service.ts:128-134).
- Guest device token ký bằng AUTH_GUEST_DEVICE_TOKEN_SECRET riêng (không phải JWT_SECRET) và deviceKey = HMAC-SHA256(secret, "device:"+deviceId) (guest-device-token.service.ts:71-75): đổi secret này làm mọi deviceKey đổi nên khoá quota theo thiết bị của matching bị tách thành key mới.
