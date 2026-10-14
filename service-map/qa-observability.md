# Quan sát và vận hành: metrics, log, trace, health, audit?

id: qa:observability
category: observability
services: module:app-shell, module:admin, module:matching

Metrics và tracing qua OpenTelemetry, log JSON bằng pino, probe liveness/readiness cho Kubernetes và audit log cho hành động nhạy cảm của admin.

### Metrics và tracing

Module định nghĩa Histogram/Meter qua @opentelemetry/api và có tracing.ts khởi tạo OpenTelemetry cho core-api.

services: matching; evidence: libs/observability/src/lib/tracing.ts:108

### Log có cấu trúc

Log JSON bằng nestjs-pino để truy vết và tìm kiếm.

services: app-shell; evidence: apps/core-api/src/app/app.module.ts:55

### Health và audit

Có endpoint liveness/readiness cho K8s; hành động của admin ghi audit log.

services: app-shell, admin; evidence: apps/core-api/src/app/health.controller.ts:14, apps/core-api/src/modules/admin/admin.service.ts:49

