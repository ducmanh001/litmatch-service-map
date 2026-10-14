# Techniques used by auth

id: techniques:auth
module: module:auth

### CSRF double-submit token (csrf-double-submit)

- how: Server phát CSRF token; client gửi lại trong header và server so với giá trị gắn ở cookie.
- why: Cookie tự đính kèm nên cần chặn request giả mạo từ site khác.
- applied here: CsrfGuard chỉ gắn ở POST /auth/refresh và /auth/logout (e5, e6); csrf_token cookie cũng httpOnly, client nhận giá trị qua JSON body rồi gửi lại header x-csrf-token (ADR 0007). Các route còn lại không dùng CSRF.
- endpoints: e5, e6
- alternative: Chỉ dựa SameSite: bỏ được token nhưng phụ thuộc trình duyệt và cấu hình khác site.

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: JwtModule.registerAsync({ global: true }) ký access token bằng JWT_SECRET; JwtAuthGuard toàn cục xác thực không trạng thái, GuestDeviceTokenService cũng dùng JwtService.
- endpoints: e1, e3, e4, e7, e8, e5, e9
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Refresh token xoay vòng + phát hiện tái sử dụng (token family) (refresh-rotation)

- how: Mỗi lần refresh phát token mới và vô hiệu token cũ; dùng lại token cũ bị coi là bị lộ và thu hồi cả họ token (family).
- why: Giới hạn thiệt hại khi lộ refresh token và phát hiện được việc tái sử dụng.
- applied here: Rotation chỉ ở POST /auth/refresh (e5): adapter rotate khoá hàng và UPDATE rotated_at IS NULL; token đã rotate bị dùng lại trả REFRESH_TOKEN_REUSED và revoke cả family. Login/upgrade chỉ phát family mới hoặc revoke.
- endpoints: e5, e1, e3, e4, e7, e8
- alternative: Refresh token cố định đến khi hết hạn: đơn giản nhưng lộ là dùng được lâu.

### OTP băm HMAC, giới hạn số lần thử (otp-hmac)

- how: Mã OTP chỉ lưu dạng băm HMAC có khoá, có hạn dùng và giới hạn số lần thử.
- why: Lộ DB cũng không đọc được mã; HMAC có khoá chống dò ngược bằng bảng băm.
- applied here: OtpService băm OTP bằng HMAC với AUTH_OTP_PEPPER, giới hạn số lần thử AUTH_OTP_MAX_ATTEMPTS; /auth/otp/request thêm @Throttle 5 lần/phút theo IP.
- endpoints: e2, e3, e7
- alternative: 

### Refresh token trong cookie httpOnly/SameSite (httponly-cookie-session)

- how: Refresh token nằm trong cookie httpOnly + SameSite nên JavaScript không đọc được.
- why: Chặn XSS đánh cắp token; đổi lại phải có CSRF bảo vệ vì cookie tự đính kèm.
- applied here: auth.cookies.ts đặt refresh token vào cookie với httpOnly: true, secure và sameSite (strict hoặc none khi cross-origin dev).
- endpoints: e5, e6
- alternative: Lưu token trong localStorage: dễ dùng nhưng JavaScript đọc được nên XSS lấy được.

### Xác minh ID token nhà cung cấp (Apple/Google/Facebook) (social-id-token)

- how: Server xác minh chữ ký và audience của ID token do Apple/Google/Facebook cấp trước khi tin danh tính.
- why: Không tin client tự khai; nhà cung cấp danh tính là bên xác thực.
- applied here: SocialVerifierService.verify: Google/Apple qua jwtVerify với createRemoteJWKSet (issuer + audience clientId), Facebook qua Graph debug_token; uid lấy từ sub/user_id làm providerUid.
- endpoints: e4, e8
- alternative: 

### Token thiết bị cho khách (HMAC) (guest-device-token)

- how: Token thiết bị cấp cho khách, ký HMAC, gắn với thiết bị để tính hạn mức.
- why: Khách tạo tài khoản vô hạn nên hạn mức phải bám thiết bị chứ không bám tài khoản.
- applied here: GuestDeviceTokenService cấp và kiểm token thiết bị cho khách, khoá thiết bị bằng createHmac('sha256') trên deviceId; AuthService.guest inject service này.
- endpoints: e1
- alternative: 

### Xác minh ID token nhà cung cấp (Apple/Google/Facebook) (social-id-token)

- how: Server xác minh chữ ký và audience của ID token do Apple/Google/Facebook cấp trước khi tin danh tính.
- why: Không tin client tự khai; nhà cung cấp danh tính là bên xác thực.
- applied here: SocialVerifierService được AuthService dùng để xác minh ID token nhà cung cấp trong luồng /auth/social và /auth/upgrade/social.
- endpoints: 
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: isGuest nằm trong payload access token và AuthSession; chính sách guest do module khác thực thi (GuestMatchQuotaService đọc isGuest từ DB, không tin JWT cũ), còn auth chỉ chặn upgrade khi user không còn guest.
- endpoints: e1, e3, e4, e5, e2, e6, e7, e8
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Guard toàn cục + @Public() để mở (global-guard-public-optout)

- how: Guard JWT đăng ký toàn cục; route nào muốn mở phải gắn @Public() tường minh.
- why: An toàn theo mặc định: quên cấu hình thì bị chặn chứ không bị mở.
- applied here: AuthController gắn @Public() ở cấp class để mở các route auth khỏi JwtAuthGuard toàn cục.
- endpoints: e1, e2, e3, e4, e5, e6
- alternative: Gắn guard từng controller: dễ sót route.

### Kiểm tra quyền sở hữu tài nguyên (ownership-check)

- how: So userId của tài nguyên với userId người gọi trước khi đọc hoặc sửa.
- why: Ngăn IDOR: không thao tác được trên tài nguyên của người khác bằng cách đoán id.
- applied here: Luồng upgrade so existing.userId !== userId, nếu identity đã gắn tài khoản khác thì ném IDENTITY_ALREADY_LINKED, không tự merge.
- endpoints: e7, e8
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: AuthService.refresh sau rotate đọc lại user từ DB (getByIdOrThrow + assertActive) và revoke token mới nếu user đã bị ban, thay vì tin trạng thái lúc phiên cũ được cấp.
- endpoints: e5
- alternative: 

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Tạo user và AuthIdentity trong this.dataSource.transaction; TypeOrmRefreshSessionAdapter.rotate cũng chạy trong một transaction.
- endpoints: e1, e3, e4, e7, e8
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: @Unique('uq_auth_identities_provider_uid') chặn đăng ký song song; service bắt isUniqueViolation rồi đọc lại identity đã tạo.
- endpoints: e1, e3, e4, e7, e8
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: AuthService truyền manager vào userService.createWithManager để tạo user cùng transaction với AuthIdentity.
- endpoints: e1, e3, e4
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Upgrade khoá User bằng lock pessimistic_write trong transaction, và adapter rotate khoá hàng RefreshToken trước khi xoay token.
- endpoints: e7, e8
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: TypeOrmRefreshSessionAdapter.rotate khoá hàng refresh_tokens (pessimistic_write) trong transaction rồi UPDATE rotated_at IS NULL, nên hai refresh song song cùng token chỉ một bên thắng, bên thua bị coi là reuse.
- endpoints: e5
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Throttle riêng: guest 5/phút, otp/request 5/phút, otp/verify, social, upgrade otp/social và search-by-phone 10/phút; refresh/logout không có @Throttle riêng, chỉ chịu ThrottlerGuard toàn cục.
- endpoints: e1, e2, e3, e4, e7, e8, e9, e5, e6
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: Controller gắn @ApiOperation và @ApiOkResponse({ type: AuthTokensDto }) để sinh hợp đồng OpenAPI cho các route auth.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9
- alternative: 

### Băm HMAC có khoá (hmac-hashing)

- how: Băm HMAC với khoá bí mật.
- why: Không dò ngược được bằng bảng băm.
- applied here: createHmac('sha256') dùng cho khoá thiết bị khách (deviceKey) và băm OTP (hash với AUTH_OTP_PEPPER).
- endpoints: e1, e2, e3, e7
- alternative: 

### So sánh hằng thời gian (constant-time-compare)

- how: So sánh hằng thời gian (timingSafeEqual).
- why: Chống tấn công đo thời gian.
- applied here: OtpService so khớp mã OTP băm bằng timingSafeEqual sau khi kiểm độ dài buffer, tránh lộ qua thời gian so sánh.
- endpoints: e3, e7
- alternative: 

### Chống dò tài khoản (anti-enumeration)

- how: Phản hồi đồng nhất để không dò được tài khoản có tồn tại hay không.
- why: Chống thu thập người dùng.
- applied here: PhoneSearchService.search trả null cả khi identity phone không tồn tại lẫn khi isSearchableByPhone tắt (và user không Active), nên /auth/search-by-phone không là oracle dò số điện thoại.
- endpoints: e9
- alternative: 

### Cô lập secret (chỉ core-api giữ khoá) (secret-isolation)

- how: Khoá và secret chỉ nằm ở core-api, không ra client.
- why: Giảm bề mặt lộ khoá: khoá ký và secret chỉ nằm ở backend, phía client không bao giờ nhận được.
- applied here: GuestDeviceTokenService ký token và HMAC deviceKey bằng AUTH_GUEST_DEVICE_TOKEN_SECRET riêng (min 32 ký tự), tách khỏi JWT_SECRET, nên token thiết bị không dùng được làm access token.
- endpoints: e1
- alternative: 

