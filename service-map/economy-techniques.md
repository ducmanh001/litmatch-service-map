# Techniques used by economy

id: techniques:economy
module: module:economy

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: JwtAuthGuard toàn cục bảo vệ các route ví, IAP, payOS, VIP; riêng webhook mở bằng @Public().
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Xác thực chữ ký/token của webhook (webhook-signature)

- how: Webhook phải được xác minh chữ ký/token của nhà cung cấp trên body gốc trước khi xử lý.
- why: Endpoint webhook công khai nên chữ ký là cách duy nhất chứng minh nguồn gửi.
- applied here: Webhook payOS có trường signature kiểm HMAC checksum, Apple nhận signedPayload JWS, Google RTDN kiểm OIDC qua các verifier thay thế được bằng env.
- endpoints: e12, e10, e11
- alternative: 

### ABAC theo thuộc tính (guest, ban, VIP, giới tính, tuổi…) (abac-attribute-policy)

- how: Quyết định dựa trên thuộc tính của người dùng hoặc tài nguyên như guest, bị ban, hạng VIP, giới tính, tuổi.
- why: Nghiệp vụ có quy tắc theo ngữ cảnh mà vai trò đơn thuần không diễn đạt được.
- applied here: assertNotGuestForPayos chặn user.isGuest nạp Diamond (PAYOS_GUEST_FORBIDDEN); gói VIP có thuộc tính tier (VipTier).
- endpoints: e4, e1, e2, e3, e5, e6, e7, e8, e9
- alternative: Policy engine (OPA/Casbin/CASL) tách chính sách khỏi mã nhưng thêm thành phần vận hành.

### Guard toàn cục + @Public() để mở (global-guard-public-optout)

- how: Guard JWT đăng ký toàn cục; route nào muốn mở phải gắn @Public() tường minh.
- why: An toàn theo mặc định: quên cấu hình thì bị chặn chứ không bị mở.
- applied here: EconomyWebhooksController gắn @Public() ở cấp class vì webhook store-to-store không có JWT, xác thực bằng chữ ký.
- endpoints: e10, e11, e12
- alternative: Gắn guard từng controller: dễ sót route.

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Unique constraint uq_transactions_idempotency_key và UQ_PAYOS_ORDER_IDEMPOTENCY_KEY cùng header ApiIdempotencyKeyHeader chặn replay mua/nạp.
- endpoints: e4, e7, e8, e10, e13, e12
- alternative: 

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: EconomyService và LedgerPersistencePort truyền EntityManager (withinTransaction) để các bước ví, ledger, outbox chung một transaction.
- endpoints: e10, e13, e12
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: Đọc Wallet với lock pessimistic_write, và job iap-refund-poll khoá receipt bằng setLock('pessimistic_write').
- endpoints: e13, e14, e12
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Transactional outbox (transactional-outbox)

- how: Sự kiện ghi vào bảng outbox trong cùng transaction nghiệp vụ; tiến trình relay đẩy ra broker rồi đánh dấu đã gửi.
- why: Tránh dual-write: không có chuyện DB commit mà event mất hoặc ngược lại.
- applied here: ledger.applyWalletDeltas ghi OutboxEvent cho mỗi thay đổi số dư cùng transaction; mua VIP thêm economy.vip.purchased, sendGift thêm economy.gift.sent; OutboxRelayService đẩy sang bus sau.
- endpoints: e8, e14
- alternative: Publish trực tiếp sau commit: nhanh nhưng mất event khi crash giữa hai bước.

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: OutboxRelayService.flushOnce giữ transaction mở suốt lúc publish; iap-refund-poll claim lô rồi commit ngay trước khi gọi API store/refund để tránh deadlock với transaction của RefundService.
- endpoints: e13, e14
- alternative: 

### FOR UPDATE SKIP LOCKED (nhiều worker không dẫm chân) (skip-locked)

- how: FOR UPDATE SKIP LOCKED lấy các hàng chưa bị worker khác khoá.
- why: Nhiều worker xử lý hàng đợi trong DB mà không dẫm chân nhau.
- applied here: Cả iap-refund-poll và outbox-relay dùng setOnLocked('skip_locked') để nhiều replica lấy lô khác nhau, không dẫm chân.
- endpoints: e13, e14
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: @Unique('uq_iap_receipts_provider_txn') chặn double-credit receipt, orderCode payOS có unique: true với sequence payos_order_code_seq.
- endpoints: e4
- alternative: 

### Advisory lock (pg_advisory) để serialize / singleton (advisory-lock)

- how: pg_advisory_xact_lock khoá theo khoá logic, không cần có hàng cụ thể, giữ đến hết transaction.
- why: Tuần tự hoá việc không gắn với một hàng (ghép đôi, worker singleton).
- applied here: ManagedInterval với clusterSingleton lấy pg_try_advisory_xact_lock trước mỗi lượt fast/deep của ReconciliationService nên chỉ một replica đối soát tại một thời điểm.
- endpoints: e15
- alternative: 

### Trạng thái suy ra từ dữ liệu gốc/thời gian (không lưu cờ) (derived-state)

- how: Trạng thái được tính lại từ dữ liệu gốc và giờ server mỗi lần đọc, không lưu cờ.
- why: Không cần timer hay job để chuyển trạng thái và không có cờ bị lệch.
- applied here: Wallet.activeVipTier là getter suy ra tier từ vipTier và vipExpiresAt > now, nên VIP hết hạn tự hạ về null khi đọc ví mà không cần job cập nhật.
- endpoints: e1
- alternative: 

### Sổ cái kép append-only (double-entry-ledger)

- how: Mỗi biến động tiền ghi ít nhất hai bút toán nợ/có append-only; số dư là snapshot dẫn xuất.
- why: Tổng nợ luôn bằng tổng có nên phát hiện và đối soát được sai lệch.
- applied here: LedgerService là writer duy nhất ghi LedgerTransaction, LedgerEntry (Nợ/Có) và LedgerAccount; số dư ví suy ra từ bút toán.
- endpoints: 
- alternative: 

### Sửa sai bằng bút toán đảo (reversal), không sửa lịch sử (reversal-compensation)

- how: Sửa sai bằng bút toán đảo mới thay vì sửa hoặc xoá lịch sử.
- why: Lịch sử luôn đầy đủ và truy vết được.
- applied here: TransactionType.Reversal cho phép LedgerService ghi giao dịch đảo (kèm idempotencyKey) khi hoàn tiền thay vì sửa lịch sử.
- endpoints: 
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: ledger_entries append-only tuyệt đối, DB trigger chặn UPDATE/DELETE (migration 1752000000000); LedgerService là writer duy nhất.
- endpoints: 
- alternative: 

### Kafka event bus (kèm DLQ) (kafka-event-bus)

- how: Event bus bền: producer ghi theo topic, consumer group đọc, lỗi lặp đưa vào DLQ.
- why: Sự kiện nghiệp vụ quan trọng cần bền và đọc lại được.
- applied here: OutboxRelayService inject EventBusPort để phát sự kiện economy ra bus (Kafka) và chuyển sang topic .DLQ khi quá giới hạn.
- endpoints: e14
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: IapRefundPollService và OutboxRelayService dùng new ManagedInterval() chạy vòng lặp định kỳ từ onApplicationBootstrap.
- endpoints: e15
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: ReconciliationService chạy fastJob với clusterSingleton: { dataSource } để chỉ một replica thực hiện đối soát tại một thời điểm.
- endpoints: e15
- alternative: 

### Hết hạn lười (xử lý khi có request) (lazy-expiry)

- how: Hết hạn được xử lý khi có request đụng tới, không cần job.
- why: Đúng đắn không phụ thuộc job; job chỉ để dọn.
- applied here: PayosService.getOrderStatus chuyển đơn Pending có expiresAt đã qua sang Expired khi đọc bằng UPDATE có điều kiện status Pending, không cần job quét; webhook đến trễ vẫn đặt Paid.
- endpoints: e5
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: Route nạp payOS 10 lần/phút và webhook apple 60 lần/phút dùng @Throttle riêng bên cạnh ThrottlerGuard toàn cục.
- endpoints: e4, e7, e10, e11, e12, e1, e2, e3, e5, e6, e8, e9
- alternative: 

### Lỗi thì chặn (fail closed) (fail-closed)

- how: Khi thành phần kiểm tra lỗi thì từ chối thay vì cho qua.
- why: An toàn hơn khi mất Redis hoặc quota.
- applied here: IapVerifier có chế độ disabled fail-closed ở production; DevIapVerifier, DevAppleNotificationVerifier, DevGoogleRtdnVerifier ném lỗi onApplicationBootstrap nếu production còn dùng dev nên app không khởi động.
- endpoints: e16, e17, e18
- alternative: 

### Dead-letter queue (dead-letter)

- how: Tin lỗi lặp lại được chuyển sang dead-letter queue để xử lý sau.
- why: Không chặn hàng đợi vì một tin hỏng.
- applied here: Quá ECONOMY_OUTBOX_MAX_ATTEMPTS, deadLetter() gọi eventBus.publishDeadLetter sang {topic}.DLQ và luôn đặt dead_lettered_at trong finally, kể cả khi publish DLQ lỗi, để thoát vòng retry.
- endpoints: e14
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: EconomyMetrics dùng OTel instruments và các method best-effort: lỗi metrics không thay đổi kết quả giao dịch tiền.
- endpoints: 
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: EconomyController gắn @ApiOperation và @ApiOkResponse (ví dụ WalletDto) cho các route ví, IAP, payOS, VIP.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: GET /economy/transactions nhận CursorPageQueryDto với @ApiCursorPageQuery và phân trang theo cursor.
- endpoints: e9
- alternative: 

### Cột JSONB (jsonb-columns)

- how: Cột JSONB lưu dữ liệu linh hoạt.
- why: Cấu trúc thay đổi thường xuyên.
- applied here: IapReceipt.rawPayload và OutboxEvent.payload là cột jsonb lưu payload thô từ store và nội dung sự kiện.
- endpoints: 
- alternative: 

### So sánh hằng thời gian (constant-time-compare)

- how: So sánh hằng thời gian (timingSafeEqual).
- why: Chống tấn công đo thời gian.
- applied here: PayosClient so chữ ký webhook bằng timingSafeEqual sau khi kiểm độ dài buffer hex.
- endpoints: 
- alternative: 

### Băm HMAC có khoá (hmac-hashing)

- how: Băm HMAC với khoá bí mật.
- why: Không dò ngược được bằng bảng băm.
- applied here: PayosClient.sign dựng canonical key=value đã sort rồi createHmac('sha256', checksumKey) để ký/kiểm payload payOS.
- endpoints: 
- alternative: 

