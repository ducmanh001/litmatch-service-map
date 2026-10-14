# Vòng story, xem story và người đã xem

id: flow:story-ring-view-and-viewers
group: Xã hội & nội dung
services: module:feed, module:friend, module:safety, module:user

Feed sở hữu Story/StoryView nhưng không sở hữu quan hệ bạn bè, block hay quyền riêng tư, nên mỗi lần liệt kê vòng story, xem một story hoặc liệt kê người xem đều phải hỏi friend, safety và user trước khi đọc hoặc ghi bảng của feed.

Ba endpoint story của feed lần lượt hỏi friend, safety, user để lọc bạn bè, block, hideProfile rồi mới đọc/ghi Story và StoryView.

## Participants

- web (client): Web client
- feed (service): Feed
- friend (service): Friend
- safety (service): Safety
- user (service): User
- postgres (infra): PostgreSQL

## Steps

1. web -> feed [call/http]: GET /stories/ring (JWT) lấy vòng story
2. feed -> friend [call/di-call]: Lấy danh sách id bạn bè của người gọi {apps/core-api/src/modules/feed/services/story.service.ts:89}
3. feed -> safety [call/di-call]: Lấy các id bị block (hai chiều) để loại khỏi vòng {apps/core-api/src/modules/feed/services/story.service.ts:90}
4. feed -> user [call/di-call]: Đọc cài đặt riêng tư để bỏ tác giả hideProfile {apps/core-api/src/modules/feed/services/story.service.ts:95}
5. feed -> postgres [call/shared-db]: Truy vấn Story còn hạn của các tác giả hợp lệ {apps/core-api/src/modules/feed/services/story.service.ts:101}
6. feed -> web [return/http]: Trả danh sách story của mình và bạn bè
7. web -> feed [call/http]: GET /stories/:storyId để xem một story
8. feed -> postgres [call/shared-db]: Đọc Story theo id, kiểm tra tồn tại và hết hạn {apps/core-api/src/modules/feed/services/story.service.ts:119}
9. feed -> safety [call/di-call]: Hỏi isBlocked theo cả hai chiều giữa người xem và tác giả (Chỉ chạy khi người xem không phải tác giả; isBlocked chiều thứ hai chỉ được hỏi nếu chiều đầu trả false, và areFriends bị bỏ qua khi story có audience public.) {apps/core-api/src/modules/feed/services/story.service.ts:129, apps/core-api/src/modules/feed/services/story.service.ts:130}
10. feed -> friend [call/di-call]: Kiểm tra audience: story không public thì phải là bạn bè {apps/core-api/src/modules/feed/services/story.service.ts:133}
11. feed -> user [call/di-call]: Hỏi tác giả có đang ẩn hồ sơ (isHidden) không {apps/core-api/src/modules/feed/services/story.service.ts:134}
12. feed -> postgres [call/shared-db]: Ghi StoryView (bỏ qua nếu tự xem hoặc đã xem trước đó) {apps/core-api/src/modules/feed/services/story.service.ts:151}
13. feed -> web [return/http]: Trả story, hoặc 404 STORY_NOT_FOUND khi bị chặn
14. web -> feed [call/http]: GET /stories/:storyId/viewers (tác giả xem người đã xem)
15. feed -> postgres [call/shared-db]: Đọc Story, chỉ tác giả được tiếp tục {apps/core-api/src/modules/feed/services/story.service.ts:168}
16. feed -> safety [call/di-call]: Lấy id bị block hiện tại để ẩn khỏi danh sách viewer {apps/core-api/src/modules/feed/services/story.service.ts:176}
17. feed -> postgres [call/shared-db]: Đọc StoryView của story theo thời gian giảm dần {apps/core-api/src/modules/feed/services/story.service.ts:177}
18. feed -> web [return/http]: Trả viewerIds đã loại người bị block

## Invariants

- Feed chỉ sở hữu Story/StoryView nên mọi quyết định xem được hay không phải hỏi friend, safety và user trước khi đọc hoặc ghi dữ liệu của feed.
- Block được lọc tại thời điểm đọc (cả ring lẫn viewers): viewer đã xem trước khi block vẫn bị ẩn khỏi danh sách nếu block xảy ra sau đó.
- Story không tồn tại, hết hạn, bị block, bị ẩn hoặc audience không cho phép đều trả cùng mã STORY_NOT_FOUND (404) nên người gọi không dò được lý do.

## Failure modes

- when Story không tồn tại, hết hạn, bị block, tác giả ẩn hồ sơ hoặc người xem không phải bạn bè của story không public: viewStory ném DomainException STORY_NOT_FOUND (404), không ghi StoryView.
- when Người xem đã có StoryView của story này (vi phạm unique): Lỗi unique bị bắt và bỏ qua nên việc ghi lượt xem là idempotent; lỗi khác vẫn được ném lại.
- when Người gọi /viewers không phải tác giả hoặc story không tồn tại: listViewers ném STORY_NOT_FOUND (404) trước khi đọc StoryView.
- when Danh sách tác giả hợp lệ rỗng (authorIds.length === 0): getRing trả mảng rỗng mà không truy vấn bảng Story; thực tế hiếm xảy ra vì userId của chính người gọi luôn nằm trong visibleIds và không bị lọc hideProfile.

## Endpoints covered

- GET /stories/ring [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- GET /stories/:storyId [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
- GET /stories/:storyId/viewers [module:feed] -> https://ducmanh001.github.io/litmatch-atlas-demo/pack/api/feed.md
