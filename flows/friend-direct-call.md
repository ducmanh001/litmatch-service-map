# Gọi trực tiếp cho bạn bè

id: flow:friend-direct-call
group: Ghép đôi & gọi
services: module:calling, module:friend, module:safety, module:user

Hai người đã follow nhau gọi voice lâu dài không qua ghép đôi: Calling phải hỏi Friend (và Safety bên trong Friend) xem được phép gọi chưa, rồi mới tạo hoặc lấy CallSession và cấp token LiveKit.

Calling kiểm quyền gọi qua Friend (block + follow hai chiều), tạo/lấy CallSession rồi ký token LiveKit trả về cho client.

## Participants

- web (client): Web client
- calling (service): Calling
- friend (service): Friend
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> calling [call/http]: POST /calling/friends/:friendUserId/join (đã qua JWT guard, throttle 20 lần/phút) (Route có @Throttle 20 request mỗi phút; mọi REST route yêu cầu JWT trừ khi @Public.)
2. calling -> friend [call/di-call]: Hỏi ProfileSocialService.canCall(userId, friendUserId) {apps/core-api/src/modules/calling/calling.service.ts:208}
3. friend -> safety [call/di-call]: Kiểm tra block theo cả hai chiều (A chặn B, B chặn A) (Hai lời gọi isBlocked chạy song song; nếu một chiều bị block thì canCall trả false ngay.) {apps/core-api/src/modules/friend/services/profile-social.service.ts:152}
4. friend -> postgres [call/shared-db]: Kiểm tra follow hai chiều còn active (areMutuallyFollowing) {apps/core-api/src/modules/friend/services/profile-social.service.ts:156}
5. friend -> calling [return/di-call]: Trả true/false; false thì Calling ném 404 SESSION_NOT_FOUND (Nhánh lỗi: thông báo 'Hai bạn cần follow nhau để gọi voice lâu dài' với HttpStatus.NOT_FOUND.)
6. calling -> postgres [call/shared-db]: Tìm CallSession kind=Friend của cặp canonical đang Pending/Active {apps/core-api/src/modules/calling/calling.service.ts:216}
7. calling -> postgres [call/shared-db]: Chưa có thì tạo CallSession Pending; trùng unique thì đọc lại bản đang có (Không có transaction: chống race bằng cách bắt lỗi unique violation rồi getOneOrFail.) {apps/core-api/src/modules/calling/calling.service.ts:228}
8. calling -> calling [call/sdk]: Ký token join room qua LivekitRoomPort.mintJoinToken (identity = userId) (Adapter dùng AccessToken của SDK với apiKey/secret, ký cục bộ; trong code này không có lời gọi mạng tới media-server ở bước cấp token.) {apps/core-api/src/modules/calling/calling.service.ts:252}
9. calling -> user [call/di-call]: Chỉ khi bật multi-region: đọc region hai bên để chọn LIVEKIT URL (Nếu chưa cấu hình LIVEKIT_REGION_URLS thì trả LIVEKIT_URL mặc định, không gọi User; lỗi lookup cũng fallback URL mặc định.) {apps/core-api/src/modules/calling/calling.service.ts:583}
10. calling -> web [return/http]: Trả JoinCallDto: call, token, livekitUrl

## Invariants

- Calling không tự đọc Friendship/follow/block: quyền gọi do Friend quyết định (canCall) và phải được hỏi trước khi tạo CallSession, nên không bao giờ có phiên hay token cho cặp bị block hoặc chưa follow nhau.
- Token LiveKit chỉ được ký sau khi đã có CallSession (roomName lấy từ call), identity do server đặt là userId của người gọi nên client không chọn được danh tính.
- CallSession friend dùng cặp canonical (userAId, userBId) nên hai chiều gọi cùng tìm ra một phiên Pending/Active; unique violation được đọc lại thay vì tạo trùng.

## Failure modes

- when Một bên block bên kia hoặc chưa follow hai chiều: canCall trả false, Calling ném DomainException 404 SESSION_NOT_FOUND và không tạo CallSession, không cấp token.
- when Hai request đồng thời cùng tạo CallSession cho một cặp: Request thua gặp unique violation, đọc lại CallSession đang có bằng getOneOrFail và dùng nó; lỗi khác unique được ném ra.
- when Lookup region của user lỗi khi multi-region bật: Calling ghi warn và fallback LIVEKIT_URL; việc chọn URL không chặn join call.

## Endpoints covered

- POST /calling/friends/:friendUserId/join [module:calling] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/calling.md
