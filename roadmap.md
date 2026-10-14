# Roadmap · Litmatch Service Map

id: roadmap
site: map
updated: 2026-10-14; repo ducmanh001/litmatch @ d3cb28c

"Roadmap" của trang này: **A. lộ trình đọc**, **B. quan sát và trạng thái xử lý** (việc còn mở của hệ thống, theo service map), **C. phạm vi phủ**, **D. lộ trình dự án** (nằm ở Atlas, có liên kết).
Tệp máy đọc: [llms.txt](llms.txt) · [sitemap.xml](sitemap.xml) · [search-index.json](search-index.json) · [index.json](index.json).

## A. Lộ trình đọc

| Câu hỏi | Mở trước | Mở tiếp |
|---|---|---|
| Service A gọi service B để làm gì, nếu B vắng thì sao | service-map/<A>.md | service-map/<B>.md |
| Kỹ thuật dùng ở một service (idempotency, outbox, lock, retry) | service-map/<service>-techniques.md | service-map/qa-idempotency.md, service-map/qa-consistency-concurrency.md |
| Một luồng xuyên service từ đầu tới cuối | flows/index.md rồi flows/<id>.md | json/flows/<id>.json (bản có cấu trúc) |
| Endpoint hoặc job này thuộc luồng nào | lookup/entrypoint-to-flow.md | flows/<id>.md |
| Câu hỏi chéo (authn, authz, rate limit, webhook, realtime, token) | service-map/qa-*.md (xem service-map/index.md) | service-map/findings.md |
| Có gì đáng lo, cái nào đã sửa | mục B của trang này | service-map/findings.md |
| Bảng, cột, API chi tiết, FE | https://ducmanh001.github.io/litmatch-atlas-demo/pack/modules/index.md | https://ducmanh001.github.io/litmatch-atlas-demo/roadmap.md |
| Tra theo tên | lookup/index.md | search.html, search-index.json |

## B. Quan sát và trạng thái xử lý

Chưa xử lý: 8/13. Còn lại đã sửa trong code (đối chiếu commit).

| id | trạng thái | service | quan sát |
|---|---|---|---|
| V1 | fixed | signaling-gateway | Relay realtime có thể nhân bản event khi chạy nhiều pod? |
| V6 | fixed | short-video | Đếm view video có race khi hai request song song? |
| V7 | fixed | short-video | Sweeper video failed có thể bỏ đói video mới? |
| V8 | confirmed | short-video | sort=ranked loại video chưa có rankScore dù comment nói fallback |
| V10 | fixed | mini-game | Hủy mini-game có thể ghi đè kết quả đã chốt? |
| V11 | fixed | economy | Kiểm bundleId Apple bị bỏ qua khi biến môi trường rỗng? |
| V12 | confirmed | safety | getHiddenUserIds đọc mọi report vĩnh viễn? |
| V13 | confirmed | matching | Lazy-expire lời mời có thể không được lưu? |
| M1 | confirmed | media-server | Chú thích đầu file k8s của media-server lỗi thời so với cấu hình |
| M2 | confirmed | matching, signaling-gateway | Sự kiện match.matched được khai báo nhưng không có nơi nào phát |
| M3 | confirmed | movie-match, signaling-gateway | Movie Match ẩn danh phát realtime bên trong transaction, trái với tài liệu |
| M4 | confirmed | auth, app-shell | AUTH_MIN_AGE và DEPLOY_ENVIRONMENT có trong .env.example nhưng không có schema hay mã đọc |
| M5 | confirmed | signaling-gateway | Đặc tả AsyncAPI trong repo chỉ liệt kê 8 trong 26 sự kiện realtime |

Chi tiết từng quan sát: [service-map/findings.md](service-map/findings.md).

## C. Phạm vi phủ

- 25 service/module, 72 luồng xuyên service (flows/index.md), 214 entrypoint (188 thuộc luồng xuyên service, 26 xử lý cục bộ trong một service, 0 chưa rõ).
- Mỗi service có datasheet (collaborator kèm "vì sao" và "nếu vắng thì sao") và thẻ kỹ thuật.
- Giới hạn: phân tích tĩnh từ code, không quan sát runtime; datasheet bảng/API/FE ở Atlas.

## D. Lộ trình dự án

Giai đoạn, thang quy mô S0-S8, ưu tiên P0-P2 và quyết định đã hoãn nằm ở [roadmap của Atlas](https://ducmanh001.github.io/litmatch-atlas-demo/roadmap.md) (mục B).

Các trang cùng bộ: [Atlas](https://ducmanh001.github.io/litmatch-atlas-demo/) · [Service Map](https://ducmanh001.github.io/litmatch-service-map/) · [Error History](https://ducmanh001.github.io/litmatch-error-history/). Roadmap dự án và việc còn mở nằm ở [roadmap của Atlas](https://ducmanh001.github.io/litmatch-atlas-demo/roadmap.md).
