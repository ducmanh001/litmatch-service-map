# Techniques used by party-room

id: techniques:party-room
module: module:party-room

### JWT bearer (access token không trạng thái) (jwt-bearer)

- how: Client gửi access token (JWT) trong header Authorization; guard xác minh chữ ký và hạn rồi gắn user vào request, server không giữ phiên.
- why: Không trạng thái nên scale ngang dễ và không phải tra DB mỗi request; đánh đổi là không thu hồi tức thì trước khi token hết hạn.
- applied here: Các route /party/rooms/* đi qua APP_GUARD JwtAuthGuard toàn cục; riêng webhook LiveKit được mở bằng @Public().
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: Session phía server (cookie + kho phiên) thu hồi được tức thì nhưng cần kho phiên dùng chung.

### Xác thực chữ ký/token của webhook (webhook-signature)

- how: Webhook phải được xác minh chữ ký/token của nhà cung cấp trên body gốc trước khi xử lý.
- why: Endpoint webhook công khai nên chữ ký là cách duy nhất chứng minh nguồn gửi.
- applied here: PartyLivekitWebhookController gọi livekit.receiveWebhook(rawBody, authHeader) để xác thực chữ ký JWT của LiveKit, sai thì ném PartyRoomErrors.WEBHOOK_INVALID.
- endpoints: e12
- alternative: 

### Capability token ngắn hạn (grant của LiveKit) (capability-token)

- how: Cấp token ngắn hạn mang sẵn quyền cụ thể (publish, subscribe) cho đúng phòng.
- why: Quyền được enforce ngay ở hệ thống media chứ không tin client.
- applied here: LivekitPartyRoomAdapter.mintJoinToken(roomName, identity, ttlSeconds) cấp token LiveKit ngắn hạn; join làm audience (canPublish=false), host/speaker được publish.
- endpoints: e6, e8, e10, e1
- alternative: 

### Server là nguồn sự thật về giá/quyền (client không quyết) (server-authoritative)

- how: Giá, số dư, trạng thái và quyền do server tính; client chỉ gửi ý định.
- why: Client có thể bị sửa nên mọi thứ liên quan tiền hoặc quyền phải do server quyết.
- applied here: Token và quyền publish suy từ membership.role trong DB (audience canPublish=false ngay từ token); room name party-{id} và livekitUrl do server sinh/snapshot, client không chọn.
- endpoints: e1, e6
- alternative: 

### Kiểm tra theo quan hệ (bạn bè, block, thành viên phòng) (rebac-relationship)

- how: Cho phép hoặc chặn dựa trên quan hệ giữa hai người: bạn bè, block, cùng phòng.
- why: Quyền trong mạng xã hội phụ thuộc quan hệ động chứ không phải vai trò cố định.
- applied here: Quyền đổi role và mời speaker dựa trên quan hệ membership: caller phải là member active có role Host của đúng phòng, nếu không ném NOT_HOST (403).
- endpoints: e8, e9
- alternative: 

### Guard toàn cục + @Public() để mở (global-guard-public-optout)

- how: Guard JWT đăng ký toàn cục; route nào muốn mở phải gắn @Public() tường minh.
- why: An toàn theo mặc định: quên cấu hình thì bị chặn chứ không bị mở.
- applied here: Webhook LiveKit gắn @Public() để bỏ qua JwtAuthGuard toàn cục, vì xác thực bằng chữ ký trong controller.
- endpoints: e12
- alternative: Gắn guard từng controller: dễ sót route.

### Transaction DB (một đơn vị công việc) (db-transaction)

- how: Gói nhiều thao tác ghi vào một transaction: cùng commit hoặc cùng rollback.
- why: Tránh trạng thái nửa vời khi lỗi giữa chừng.
- applied here: Tạo phòng, join, leave, đổi role, speaker-invite chạy trong this.dataSource.transaction(async (manager) => ...) để room và membership đổi nhất quán.
- endpoints: e1, e6, e7, e8, e9, e10, e11, e5, e23, e12
- alternative: 

### Khoá bi quan (SELECT … FOR UPDATE) (pessimistic-lock)

- how: SELECT … FOR UPDATE khoá hàng đang xử lý để giao dịch khác phải chờ.
- why: Chặn hai request song song cùng đọc rồi cùng ghi (race) trên số dư, ticket, session.
- applied here: lockActiveRoom và các findOne lock pessimistic_write trên dòng party_rooms serialize join, leave, đổi role, invite/accept/decline, gửi comment, closeRoomById và webhook disconnect, nên đếm member/speaker và đóng phòng không race.
- endpoints: e7, e23, e1, e12, e5, e6, e8, e9, e10, e11
- alternative: Khoá lạc quan (version) ít chờ hơn nhưng phải retry khi xung đột.

### Truyền EntityManager giữa module (chung một transaction) (entity-manager-propagation)

- how: Module sở hữu dữ liệu nhận EntityManager của giao dịch đang chạy để việc của nó nằm trong cùng transaction.
- why: Nhiều module dùng chung DB nên atomicity xuyên module không cần saga hay 2PC.
- applied here: PartyRoomTransitionHook (manager, room) chỉ được closeRoomById gọi trong transaction đóng phòng; AdminService dùng để ghi audit log cùng transaction, không áp dụng cho join, role hay comment.
- endpoints: e5, e6, e8, e9, e10, e11
- alternative: Saga/outbox xuyên service: cần khi tách deployable, thừa khi cùng process.

### Máy trạng thái tường minh (state-machine)

- how: Trạng thái và các bước chuyển hợp lệ được định nghĩa tường minh.
- why: Chặn chuyển trạng thái sai thứ tự và dễ kiểm thử.
- applied here: PartyRoom đi Active -> Closed kèm closeReason (host_left, finished, swept, error, admin_closed); closeRoomById idempotent, phòng đã Closed trả closed=false nên không dọn SFU/publish lần hai.
- endpoints: e1, e7, e12, e23
- alternative: 

### Ràng buộc unique ở DB làm chốt chặn cuối (unique-constraint-guard)

- how: Ràng buộc unique hoặc index duy nhất ở DB là chốt chặn cuối cho trùng lặp.
- why: Check-rồi-insert ở tầng code vẫn dính race; DB mới bảo đảm tuyệt đối.
- applied here: Dùng isUniqueViolation/violatedConstraint từ postgres-errors để bắt vi phạm unique (UQ_PARTY_MEMBERS_ACTIVE_USER, idempotency comment) và xử lý như replay.
- endpoints: e1, e6, e5
- alternative: 

### Sửa sai bằng bút toán đảo (reversal), không sửa lịch sử (reversal-compensation)

- how: Sửa sai bằng bút toán đảo mới thay vì sửa hoặc xoá lịch sử.
- why: Lịch sử luôn đầy đủ và truy vết được.
- applied here: updateParticipantPublish chạy ngoài transaction; nếu LiveKit lỗi, syncPublishPermissionAfterCommit mở transaction mới khôi phục previousRole/previousInvitePending (chỉ khi chưa có chuyển role mới hơn) rồi trả 503.
- endpoints: e8, e10
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: party_room_comments chỉ insert, không update/xoá theo vòng đời LiveKit; seq tăng dần do DB cấp làm thứ tự ổn định cho cursor và bằng chứng Trust & Safety.
- endpoints: e4, e5
- alternative: 

### Idempotency-Key (client sinh, server khử trùng) (idempotency-key)

- how: Client gửi khoá duy nhất cho mỗi thao tác; server lưu khoá (unique) và trả lại kết quả cũ khi gặp lại.
- why: Retry mạng không tạo bản ghi hoặc trừ tiền hai lần.
- applied here: Chỉ POST comment bắt buộc Idempotency-Key; key lưu dạng party:comment:{userId}:{roomId}:{key}, vi phạm uq_party_room_comments_idempotency_key thì trả lại comment cũ nếu cùng room/sender/content, khác thì 409.
- endpoints: e5
- alternative: 

### Bảng chỉ thêm (bất biến) (append-only)

- how: Bảng chỉ thêm, không sửa hoặc xoá dòng cũ.
- why: Giữ bằng chứng bất biến cho kiểm toán và xử lý báo cáo.
- applied here: party_room_comments chỉ insert, không xoá theo vòng đời room, giữ thứ tự ổn định và bằng chứng cho Trust & Safety; unique idempotency ở DB.
- endpoints: 
- alternative: 

### Chỉ publish sau khi commit (publish-after-commit)

- how: Chỉ phát realtime sau khi transaction commit.
- why: Không bắn sự kiện của giao dịch bị rollback.
- applied here: Sự kiện như PartyMemberLeft được publishToRoomMembers sau khi transaction commit, replay comment không bắn lại event.
- endpoints: e7, e8, e10, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22
- alternative: 

### REST polling làm fallback cho realtime best-effort (polling-fallback)

- how: Client vẫn đọc trạng thái qua REST khi realtime rớt.
- why: Realtime best-effort nên cần đường dự phòng đúng đắn.
- applied here: Web refetch GET /party/rooms/:id và /comments mỗi 30s khi socket khoẻ, 5s khi mất realtime, vì event party.* chỉ best-effort và REST là nguồn sự thật.
- endpoints: e3, e4
- alternative: 

### Timer ân hạn trước khi đóng/kết thúc (grace-timer)

- how: Hẹn giờ ân hạn trước khi đóng hoặc kết thúc (ví dụ host rớt mạng).
- why: Tránh đóng phòng vì mất kết nối thoáng qua.
- applied here: Host/member rớt chỉ bị đánh dấu hostDisconnectedAt/disconnectedAt; job 5s đóng sau PARTY_HOST_DISCONNECT_GRACE_SECONDS (15s mặc định) hoặc chốt member sau PARTY_MEMBER_DISCONNECT_GRACE_SECONDS; join lại trong grace xoá cờ.
- endpoints: e6, e12, e23
- alternative: 

### Worker singleton nhờ advisory lock (singleton-worker-lock)

- how: Advisory lock bảo đảm chỉ một replica chạy worker tại một thời điểm.
- why: Tránh chạy trùng khi scale nhiều pod.
- applied here: Hai ManagedInterval của sweeper truyền clusterSingleton: { dataSource }, nên mỗi tick chỉ một replica giữ pg_try_advisory_xact_lock litmatch:cluster-job:{jobName}; replica khác rollback và chờ followerRetry.
- endpoints: e23
- alternative: 

### Sweeper dọn dữ liệu hết hạn (sweeper)

- how: Job quét dọn bản ghi hết hạn.
- why: Giải phóng tài nguyên và tránh ghi rác.
- applied here: sweepStaleRooms quét tối đa 100 phòng active quá PARTY_STALE_ROOM_SECONDS, đóng (swept) nếu không còn member hoặc roomExists trên SFU trả false; SFU lỗi thì bỏ qua, không đóng oan.
- endpoints: e23
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: PartyRoomSweeperService.onApplicationBootstrap khởi hai ManagedInterval: sweeperJob (PARTY_SWEEPER_INTERVAL_MS, 30s) và hostGraceJob (PARTY_HOST_GRACE_CHECK_INTERVAL_MS, 5s), dừng ở onApplicationShutdown.
- endpoints: e23
- alternative: 

### Sweeper dọn dữ liệu hết hạn (sweeper)

- how: Job quét dọn bản ghi hết hạn.
- why: Giải phóng tài nguyên và tránh ghi rác.
- applied here: PartyRoomSweeperService đối chiếu DB với SFU để đóng phòng rỗng hoặc SFU đã tự đóng khi webhook rớt.
- endpoints: 
- alternative: 

### Worker lặp (interval / ManagedInterval) (interval-worker)

- how: Vòng lặp theo chu kỳ trong process (ManagedInterval).
- why: Worker nền đơn giản, không cần hạ tầng queue riêng.
- applied here: PartyRoomSweeperService giữ hai ManagedInterval: sweeperJob quét phòng và hostGraceJob xử lý host rớt quá thời gian ân hạn.
- endpoints: 
- alternative: 

### Rate limit (@nestjs/throttler) (rate-limit-throttler)

- how: @Throttle giới hạn số request theo khoảng thời gian cho từng route.
- why: Chặn spam và dò quét bằng cách giới hạn số request theo thời gian, trước khi request tới logic nghiệp vụ.
- applied here: @Throttle riêng trên tạo phòng 5/phút, gửi comment 60/phút, join 20/phút; các route còn lại chỉ chịu ThrottlerGuard APP_GUARD toàn cục với hạn mức mặc định.
- endpoints: e1, e5, e6, e2, e3, e4, e7, e8, e9, e10, e11, e12
- alternative: 

### Best-effort: lỗi phụ không phá nghiệp vụ chính (best-effort-degrade)

- how: Việc phụ (push, realtime) lỗi chỉ ghi log, không phá nghiệp vụ chính.
- why: Nghiệp vụ đã commit không nên hỏng vì kênh phụ.
- applied here: Kick participant và deleteRoom trên SFU là cleanup best-effort, không giữ event realtime phía sau media timeout (publishToRoomMembers ngay sau commit).
- endpoints: e7, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22
- alternative: 

### Hợp đồng OpenAPI + client sinh tự động (openapi-contract)

- how: OpenAPI sinh từ code và client TypeScript sinh tự động.
- why: Client và server không lệch hợp đồng.
- applied here: PartyRoomController dùng @ApiOperation, @ApiOkResponse({ type: JoinPartyRoomDto }) để mô tả hợp đồng OpenAPI từng route.
- endpoints: e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: listRooms phân trang theo cặp (created_at, id) giảm dần; listComments đi lùi theo seq bigint do DB cấp, cursor decode và kiểm tra, sai thì CURSOR_INVALID (400).
- endpoints: e2, e4
- alternative: 

### Phân trang cursor (keyset) (cursor-pagination)

- how: Phân trang theo cursor thay vì offset.
- why: Ổn định khi dữ liệu thay đổi và nhanh ở trang sâu.
- applied here: GET /party/rooms dùng ListPartyRoomsQueryDto extends CursorPageQueryDto để phân trang cursor, kèm lọc tên và category.
- endpoints: 
- alternative: 

