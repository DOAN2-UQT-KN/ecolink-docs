# api-gateway

> Nguồn: `ecolink-server/api-gateway/src/index.ts` (toàn bộ logic), `ecolink-server/api-gateway/src/tracer.ts` (Datadog tracer).

## 1. Trách nhiệm

- Đây là điểm vào HTTP duy nhất cho client, cổng mặc định `8081` (`PORT`).
- Chỉ **proxy theo tiền tố path** tới 5 service upstream, dùng `express-http-proxy`.
- Gom tài liệu OpenAPI của các service thành một Swagger UI tại `/api-docs`.
- **Gateway không xác thực, không phân quyền, không rate limit.** Việc kiểm tra JWT, role và API key nội bộ do từng service tự làm (xem [05-permissions.md](../05-permissions.md)).

## 2. Cấu trúc

| File | Vai trò |
|---|---|
| `src/index.ts` | Cấu hình express, CORS, helmet, bảng proxy, error handler |
| `src/tracer.ts` | Khởi tạo Datadog APM (`dd-trace`), được import đầu tiên |

## 3. Middleware

| Middleware | Cấu hình | Bằng chứng |
|---|---|---|
| `cors` | `origin: "*"`, `credentials: true` | `src/index.ts` |
| `helmet` | `contentSecurityPolicy: false` | `src/index.ts` |
| Error handler cuối chuỗi | Mọi lỗi được trả về **502** `{ error: "Bad Gateway", message: "The upstream service is unavailable" }` | `src/index.ts` |

## 4. Endpoint

Gateway có 6 endpoint riêng, còn lại đều là proxy.

| Method | Path | Mô tả | Handler |
|---|---|---|---|
| GET | `/health` | Trả `{ status: "UP", service: "api-gateway" }` | `src/index.ts` (inline) |
| GET | `/api-docs/specs/identity.json` | Lấy `IDENTITY_SERVICE_URL/openapi.json` và ghi đè `servers` thành `GATEWAY_PUBLIC_URL` | `serveRewrittenOpenApi()` |
| GET | `/api-docs/specs/incident.json` | Như trên, cho incident-service | `serveRewrittenOpenApi()` |
| GET | `/api-docs/specs/reward.json` | Như trên, cho reward-service | `serveRewrittenOpenApi()` |
| GET | `/api-docs/specs/ai.json` | Như trên, cho ai-service | `serveRewrittenOpenApi()` |
| GET | `/api-docs` | Swagger UI gộp 4 spec | `mountGatewaySwaggerUi()` (`shared/express-swagger`) |

Mã lỗi của `serveRewrittenOpenApi()`: trả **502** `Upstream OpenAPI unreachable` khi fetch lỗi, và **502** `Upstream OpenAPI unavailable` khi upstream trả mã khác 2xx.

### Bảng proxy

Mọi method (`app.use`) được chuyển tiếp nguyên path. Trừ 2 trường hợp đánh dấu ⚠, upstream path giống hệt path ở gateway.

| Tiền tố ở gateway | Upstream | Path upstream |
|---|---|---|
| `/api/v1/auth` | identity-service (`IDENTITY_SERVICE_URL`, mặc định `:4000`) | giữ nguyên |
| `/api/v1/users` | identity-service | giữ nguyên |
| `/api/v1/roles` | identity-service | giữ nguyên |
| `/api/v1/reports` | incident-service (`INCIDENT_SERVICE_URL`, `:3001`) | giữ nguyên |
| `/api/v1/campaigns` | incident-service | giữ nguyên |
| `/api/v1/organizations` | incident-service | giữ nguyên |
| `/api/v1/organization-applications` | incident-service | giữ nguyên |
| `/api/v1/admin/organization-applications` | incident-service | giữ nguyên |
| `/api/v1/admin/media` | incident-service | giữ nguyên |
| `/api/v1/incident/votes` | incident-service | giữ nguyên |
| `/api/v1/incident/saved-resources` | incident-service | ⚠ `/incident/saved-resources…` (bỏ `/api/v1`) |
| `/api/v1/sos` | incident-service | giữ nguyên |
| `/api/v1/notifications` | notification-service (`NOTIFICATION_SERVICE_URL`, `:3003`) | giữ nguyên |
| `/api/v1/gifts`, `/api/v1/difficulties`, `/api/v1/me/points`, `/api/v1/me/redemptions`, `/api/v1/admin/gift-redemptions`, `/api/v1/leaderboard` | reward-service (`REWARD_SERVICE_URL`, `:3002`) | giữ nguyên |
| `/api/v1/seasons`, `/api/v1/me/gamification`, `/api/v1/me/badges`, `/api/v1/metric-tables`, `/api/v1/metric-columns`, `/api/v1/gamification`, `/api/v1/admin/gamification`, `/api/v1/admin/seasons` | reward-service | giữ nguyên |
| `/api/v1/chat` | ai-service (`AI_SERVICE_URL`, `:3004`) | giữ nguyên, `parseReqBody: false` (để stream SSE) |
| `/api/v1/translate` | ai-service | ⚠ `/api/v1/chat/translate…` |

Các route `/internal/v1/...` của các service **không** được gateway proxy. Chúng chỉ dùng cho gọi server-to-server qua API key.

## 5. Event / Job

Không có.

## 6. Job nền

Không có.

## 7. Phụ thuộc

- identity, incident, notification, reward, ai-service (qua HTTP).
- Datadog Agent (APM), cấu hình trong `src/tracer.ts`.
- `@da2/express-swagger` (`ecolink-server/shared/express-swagger`).

## 8. Biến môi trường

| Biến | Ý nghĩa | Mặc định trong code |
|---|---|---|
| `PORT` | Cổng lắng nghe | `8081` |
| `IDENTITY_SERVICE_URL` | URL identity-service | `http://localhost:4000` |
| `INCIDENT_SERVICE_URL` | URL incident-service | `http://localhost:3001` |
| `REWARD_SERVICE_URL` | URL reward-service | `http://localhost:3002` |
| `NOTIFICATION_SERVICE_URL` | URL notification-service | `http://localhost:3003` |
| `AI_SERVICE_URL` | URL ai-service | `http://localhost:3004` |
| `GATEWAY_PUBLIC_URL` | URL công khai, ghi vào `servers` của OpenAPI | `http://localhost:${port}` |

File `api-gateway/.env.example` hiện đang trống.

## 9. Vấn đề cần xác nhận

1. **CORS `origin: "*"` đi cùng `credentials: true`.** Trình duyệt không chấp nhận tổ hợp này khi request có gửi cookie. Cần xác nhận client có dùng cookie qua gateway không (xem 06-frontend.md).
2. **Error handler luôn trả 502** kể cả khi lỗi không đến từ upstream (ví dụ lỗi trong `serveRewrittenOpenApi`). Hệ quả là thông điệp lỗi có thể gây hiểu nhầm.
3. **Không có rate limit và giới hạn kích thước body ở gateway.** Việc bảo vệ phụ thuộc hoàn toàn vào từng service.
4. **`/api/v1/incident/saved-resources` được rewrite thành `/incident/saved-resources`**, khác với quy ước `/api/v1/...` của các route khác. Cần xác nhận đây là chủ ý (khớp router của incident-service).
5. **Notification-service không có spec trong Swagger UI gộp** (chỉ có Identity, Incident, Reward, AI).
6. `.env.example` trống, chưa liệt kê các biến ở mục 8.
