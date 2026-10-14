# Hợp đồng API và dữ liệu: OpenAPI, validate, phân trang, schema?

id: qa:api-contract
category: data
services: module:matching, module:feed, module:admin

Hợp đồng REST sinh từ chính controller (OpenAPI) rồi sinh client TypeScript; DTO validate bằng class-validator; phân trang bằng cursor; schema DB chỉ đổi bằng migration (synchronize tắt).

### OpenAPI và client sinh

Mỗi route mô tả bằng @ApiOperation; core-api xuất openapi/core-api.json và client TypeScript được sinh từ đó nên frontend không lệch hợp đồng.

services: matching, feed, admin; evidence: apps/core-api/src/modules/matching/controllers/invite.controller.ts:54

### Validate và phân trang

Dữ liệu vào được validate bằng class-validator ở biên; danh sách phân trang bằng cursor (keyset) thay vì offset.

services: matching, feed; evidence: apps/core-api/src/modules/matching/controllers/invite.controller.ts:74, apps/core-api/src/main.ts:80

### Schema chỉ qua migration

synchronize luôn tắt; mọi thay đổi schema đi qua migration có version.

services: admin; evidence: apps/core-api/src/app/app.module.ts:81

## Alternatives not chosen

- Phân trang offset: Lệch khi dữ liệu thêm hoặc xoá giữa hai trang và chậm ở trang sâu.
- synchronize tự đồng bộ schema: Có thể xoá hoặc đổi cột ngoài ý muốn trên production; docs/03 § 3.10 yêu cầu schema do migration sở hữu.
