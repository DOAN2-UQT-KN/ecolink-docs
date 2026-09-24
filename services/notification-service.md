# notification-service

> Tài liệu viết hoàn toàn từ code. Đường dẫn bằng chứng tính từ `/Users/ngoc/ecolink`.
> Viết tắt: `NS` = `ecolink-server/services/notification-service`, `IS` = `ecolink-server/services/incident-service`.

## 1. Trách nhiệm của service

- Nhận **job gửi thông báo** từ service khác qua HTTP nội bộ (API key), ghi vào bảng outbox `notification_jobs` rồi đẩy lên **AWS SQS**; worker đọc SQS, render template Handlebars và giao theo kênh.
  Bằng chứng: `NS/src/modules/notification/notification.controller.ts > notificationController.enqueue()`, `ecolink-server/shared/da2-queue/src/infra/sqs-background-job-queue.ts > SqsBackgroundJobQueue.enqueue()`.
- **Hai kênh gửi**, không có gì khác:
  - `email`: gửi bằng SMTP (nodemailer) và lưu một bản ghi `Notification` với `type = EMAIL`. `NS/src/channels/email/email.channel.ts > EmailNotificationChannel.deliver()`
  - `website` (in-app; cũng nhận `in-app`, `inapp` ở tầng worker): chỉ lưu bản ghi `Notification` với `type = WEBSITE`, kèm nội dung song ngữ en/vi trong `payload.locales`. `NS/src/channels/in-app/in-app.channel.ts > InAppNotificationChannel.deliver()`
  - **Không có push notification** (không thấy FCM/APNs/web-push trong code hay dependencies, xem `NS/package.json`). Không có websocket/SSE realtime: client phải gọi lại `GET /my` để lấy thông báo mới.
- Cho người dùng đã đăng nhập **xem danh sách thông báo in-app** và **đánh dấu một thông báo đã đọc**.
- **Không** quản lý tuỳ chọn nhận thông báo (preferences). Preferences lưu ở identity-service (`User.notificationPreferences`); **bên gọi** (incident-service) tự lọc trước khi gọi NS (xem mục 5).

## 2. Cấu trúc thư mục / module chính

```
notification-service/
├─ prisma/schema.prisma            # Notification, NotificationJob, enum NotificationType/NotificationKind
├─ prisma/migrations/              # 16 migration (init → org_application_kinds)
├─ templates/notifications/<KIND>/ # Handlebars: website.title|body[.vi].hbs, email.subject|text|html[.vi].hbs
├─ src/index.ts                    # Express app (API) — đồng thời import "./worker" nên API process cũng chạy worker
├─ src/worker.ts                   # entry của worker: startAllQueues()
├─ src/tracer.ts                   # dd-trace (Datadog APM)
├─ src/middleware/                 # auth.middleware (JWT), internal-auth.middleware (API key), error.middleware
├─ src/modules/notification/       # routes, controller, service, processor, dto, direct-email-kinds
├─ src/modules/templates/          # notification-template.engine.ts (Handlebars, cache compile)
├─ src/channels/                   # strategy + factory; email/ (channel, service, provider SMTP); in-app/
├─ src/queue/                      # register (QueueRunner), SQS factory, job store Prisma, workers/notification-send.worker.ts
├─ src/lib/                        # prisma client, identity-user.client (lấy email theo userId)
└─ src/openapi/route-models.ts     # map DTO cho Swagger
```

Không có file seed trong service này.

## 3. Cơ chế xác thực & middleware

| Cơ chế | Cách hoạt động | Bằng chứng |
|---|---|---|
| JWT người dùng | Lấy token từ header `Authorization: Bearer <token>`, nếu không có thì đọc cookie `accessToken`. Verify bằng `JWT_SECRET` (phải trùng identity-service). Payload dùng: `userId`, `email`, `role?`. Thiếu token → `401 TOKEN_MISSING`; verify lỗi → `401 TOKEN_INVALID`. **Không kiểm tra role** ở bất kỳ route nào. | `NS/src/middleware/auth.middleware.ts > authenticate()`, `NS/src/utils/jwt.utils.ts > verifyToken()` |
| API key nội bộ | Header `x-internal-api-key` so sánh `!==` với `INTERNAL_NOTIFICATION_API_KEY`. Env chưa cấu hình → `500 INTERNAL_SERVER_ERROR` "INTERNAL_NOTIFICATION_API_KEY is not configured"; sai/thiếu → `401` "Invalid internal API key". | `NS/src/middleware/internal-auth.middleware.ts > requireInternalApiKey()` |
| Kiểm tra lúc khởi động | `JWT_SECRET` rỗng → throw khi load module (service không chạy). | `NS/src/utils/jwt.utils.ts` (top-level) |
| Middleware chung | `helmet` (tắt CSP), `cors` (origin `CORS_ORIGIN` hoặc `*`, `credentials: true`), `express.json`, `urlencoded`, `cookie-parser`, `errorHandler` trả 500 kèm `stack` khi `NODE_ENV=development`. | `NS/src/index.ts`, `NS/src/middleware/error.middleware.ts > errorHandler()` |
| Gọi ra identity-service | Header `x-internal-api-key: INTERNAL_IDENTITY_API_KEY`. | `NS/src/lib/identity-user.client.ts > fetchUserEmailById()` |

## 4. Danh sách API endpoint

**Tổng: 4 endpoint** (3 nghiệp vụ dưới `/api/v1/notifications` + `GET /health`). Ngoài ra `mountOpenApi` gắn thêm `GET /openapi.json` và `/api-docs` (Swagger UI) — không tính là API nghiệp vụ (`ecolink-server/shared/express-swagger/src/index.ts`).

Gateway proxy toàn bộ prefix `/api/v1/notifications` → NS (`ecolink-server/api-gateway/src/index.ts`). Nghĩa là **`POST /api/v1/notifications/jobs` cũng truy cập được từ bên ngoài qua gateway**, dù comment trong gateway ghi "internal jobs hit service directly with API key". Route này vẫn được bảo vệ bằng API key.

### Module notification

| Method | Path (qua gateway) | Auth | Role được phép | Request (validation) | Response | Mã lỗi | Handler (file > hàm) |
|---|---|---|---|---|---|---|---|
| POST | `/api/v1/notifications/jobs` | `x-internal-api-key` | Service nội bộ (không có khái niệm role) | Body: `type` string, không rỗng (controller chỉ hiểu `email` / `website`); `kind` thuộc enum `NotificationKind`; `userId` UUID (tuỳ chọn); `payload` object (tuỳ chọn, giá trị sẽ được ép về string). Rule thêm: `website` bắt buộc `userId`; `email` + kind thuộc `KINDS_ALLOWING_DIRECT_TO_EMAIL` bắt buộc `payload.toEmail`; `email` + kind khác bắt buộc `userId`. | `202` `{ success, data: { accepted: true } }` | `400 VALIDATION_ERROR` (express-validator); `400` "userId is required for website notifications"; `400` "payload.toEmail is required for <kind>"; `400` "userId is required for email notifications"; `400` với message lỗi khi enqueue SQS thất bại; `401` / `500` từ middleware API key | `NS/src/modules/notification/notification.routes.ts` → `notification.controller.ts > notificationController.enqueue()` |
| GET | `/api/v1/notifications/my` | JWT (Bearer hoặc cookie) | Mọi user đã đăng nhập | Query: `unreadOnly` ∈ {`true`,`false`,`0`,`1`} (tuỳ chọn); `limit` int 1..200 (tuỳ chọn, mặc định 50) | `200` `{ items: NotificationItemData[], unreadCount }`. Chỉ trả thông báo `type = WEBSITE` của chính user, sắp xếp `createdAt desc`. `unreadCount` luôn đếm toàn bộ WEBSITE chưa đọc (không phụ thuộc `unreadOnly`/`limit`). | `400 VALIDATION_ERROR`; `401 TOKEN_MISSING` / `TOKEN_INVALID`; `401` nếu token không có `userId` | `notification.controller.ts > notificationController.listMine()` → `notification.service.ts > NotificationService.listForUser()` |
| PATCH | `/api/v1/notifications/:id/read` | JWT | Chủ sở hữu thông báo | Param `id` UUID | `200` `NotificationItemData` (đã có `readAt`) | `400 VALIDATION_ERROR`; `401`; `404 NOT_FOUND` nếu không tìm thấy thông báo WEBSITE có `id` và `userId` khớp | `notification.controller.ts > notificationController.markRead()` → `notification.service.ts > NotificationService.markRead()` |
| GET | `/health` (gateway **không** proxy) | Không | — | — | `200` `{ status: "ok", service: "notification-service" }` | — | `NS/src/index.ts` (inline) |

`NotificationItemData` = `{ id, userId, type, kind, title, body, payload, readAt (ISO | null), createdAt (ISO) }`. **`htmlBody` không được trả về** (`notification.controller.ts > dtoFromRow()`).

Không có: đánh dấu tất cả đã đọc, xoá thông báo, xem danh sách thông báo email, endpoint preferences.

## 5. Event/Job phát ra và lắng nghe

### 5.1 Job nội bộ (SQS)

| Tên job | Payload | Phát khi | Nhận |
|---|---|---|---|
| `SEND_NOTIFICATION` (`NotificationJobType.SEND_NOTIFICATION`) | Envelope `{ jobId, version: 1, jobType, createdAt, payload }`, trong đó `payload: SendNotificationJobPayload = { type: "email" \| "website", kind, userId?, payload?: Record<string,string> }` | `POST /jobs` hợp lệ | `NotificationSendWorker` trên queue `SQS_NOTIFICATION_QUEUE_URL` |

Bằng chứng: `NS/src/queue/notification-job.types.ts`, `NS/src/queue/register.ts`, `NS/src/queue/workers/notification-send.worker.ts > NotificationSendWorker.process()`.

Luồng outbox: `createJob` ghi `notification_jobs` với status `12 (_STATUS_PENDING)` → gửi SQS → `markEnqueued` (vẫn 12). Nếu gửi SQS lỗi → `markFailedWithoutSend` (status `23 FAILED`) rồi ném lỗi cho controller (trả 400). Không có relay nào quét lại các job FAILED/PENDING (`NS/src/queue/background-job-store.ts`).

### 5.2 HTTP gọi ra ngoài

| Đích | Khi nào | Bằng chứng |
|---|---|---|
| identity-service `GET /internal/v1/users/:userId/email` (header `x-internal-api-key`) | Job `email` không có `payload.toEmail` → cần tra email theo `userId` | `NS/src/lib/identity-user.client.ts > fetchUserEmailById()` |
| SMTP server | Mỗi job `email` | `NS/src/channels/email/email.provider.ts > EmailProvider.send()` |

### 5.3 Ai gọi vào notification-service (các "trigger thông báo")

Chỉ **incident-service** gọi `POST /api/v1/notifications/jobs` (grep toàn repo: identity-service và reward-service không gọi). Mọi lời gọi đi qua circuit breaker `HTTP_CIRCUIT_NOTIFICATION = "http->notification"`, timeout 10s, và nếu thiếu `NOTIFICATION_SERVICE_URL` hoặc `INTERNAL_NOTIFICATION_API_KEY` thì **chỉ log warning rồi bỏ qua** (`IS/src/resilience/http-circuit.ts`).

Lọc theo preferences: các client dùng `enqueueWebsiteNotificationsToUsers()` / `postReportWebsiteNotification()` gọi trước `IS/src/modules/organization/identity-user.client.ts > filterUserIdsForNotificationKind()` → identity `POST /internal/v1/users/notification-prefs/filter` (tối đa 500 id/lần). Nếu identity lỗi hoặc thiếu env thì **trả lại toàn bộ userIds** (fail-open). Luật map kind → preference key nằm ở `ecolink-server/shared/da2-constants/src/notification-preferences.ts > notificationKindToPreferenceKey()`.

| # | Kind | Kênh | Người nhận | Trigger (file > hàm) | Client gọi NS | Lọc preference |
|---|---|---|---|---|---|---|
| 1 | `ORGANIZATION_CONTACT_VERIFY` | email (`toEmail`) | Email liên hệ của tổ chức | `IS/src/modules/organization/organization.service.ts > createOrganization()`, `updateOrganization()`, `resendOrganizationContactVerificationEmail()` → `queueOrganizationContactVerificationEmail()` | `IS/src/modules/organization/organization-contact-email-notify.client.ts > enqueueOrganizationContactVerificationEmail()` | Không |
| 2 | `ORGANIZATION_APPROVED` | website | Owner tổ chức (nếu đã có `ownerId`) | `organization.service.ts > adminVerifyOrganization()` → `notifyOwnerOfOrganizationVerified("approved")` | `IS/src/modules/campaign/notification-jobs.client.ts > enqueueOrganizationApprovedWebsiteNotification()` | Có gọi filter, nhưng kind là admin-only → luôn bật |
| 3 | `ORGANIZATION_REJECTED` | website | Owner tổ chức | `adminVerifyOrganization()` → `notifyOwnerOfOrganizationVerified("banned", reason)` | `notification-jobs.client.ts > enqueueOrganizationRejectedWebsiteNotification()` | Như trên |
| 4 | `VOLUNTEER_REQUEST` | website | Owner tổ chức | `organization.service.ts > createJoinRequest()` → `notifyOrganizationOwnerOfJoinRequest()` | `notification-jobs.client.ts > enqueueVolunteerRequestWebsiteNotification()` | `volunteerRequest` |
| 5 | `VOLUNTEER_REQUEST` | website | Các manager của chiến dịch (trừ chính người xin) | `IS/src/modules/campaign/campaign_joining_request/campaign_joining_request.service.ts > createJoinRequest()` → `notifyCampaignManagersOfVolunteerRequest()` | như trên | `volunteerRequest` |
| 6 | `VOLUNTEER_APPROVED` / `VOLUNTEER_REJECTED` | website | Người xin tham gia (tổ chức) | `organization.service.ts > processJoinRequest()` | `enqueueVolunteerApprovedWebsiteNotification()` / `enqueueVolunteerRejectedWebsiteNotification()` | `volunteerRequest` |
| 7 | `VOLUNTEER_APPROVED` / `VOLUNTEER_REJECTED` | website | Người xin tham gia (chiến dịch) | `campaign_joining_request.service.ts > processJoinRequest()` | như trên | `volunteerRequest` |
| 8 | `CAMPAIGN_CREATED` | website | Thành viên active của tổ chức, trừ người tạo | `IS/src/modules/campaign/campaign.service.ts > createCampaign()` → `notifyOrganizationMembersOfNewCampaign()` | `enqueueWebsiteNotificationsToUsers()` | `campaignNew` |
| 9 | `CAMPAIGN_VERIFY_INVITE` | website | Người dân trong bán kính 5 000 m (vị trí đã lưu ở identity + người từng gửi report gần đó), trừ admin duyệt, người tạo, manager | `campaign.service.ts > adminVerifyCampaign()` → `notifyNearbyCitizensToJoinApprovedCampaign()` → `notifyNearbyCitizensForCampaignVerify()` | `enqueueWebsiteNotificationsToUsers()` | `campaignNearbyVerify` |
| 10 | `CAMPAIGN_COMPLETION_PENDING_ADMIN` | website | Danh sách user id trong env `CAMPAIGN_COMPLETION_ADMIN_NOTIFY_USER_IDS` của IS | `campaign.service.ts > submitCampaignCompletionForAdminApproval()` → `notifyAdminsCampaignCompletionPendingApproval()` | `enqueueCampaignCompletionPendingAdminWebsiteNotification()` → `postWebsiteNotificationJob()` | **Không** (gọi thẳng, bỏ qua filter) |
| 11 | `CAMPAIGN_COMPLETION_VERIFY_INVITE` | website | Người dân gần chiến dịch (như #9) | `submitCampaignCompletionForAdminApproval()` → `notifyNearbyOnCampaignCompletionSubmitted()` → `notifyNearbyCitizensForCampaignVerify()` | `enqueueWebsiteNotificationsToUsers()` | `campaignNearbyVerify` |
| 12 | `CAMPAIGN_DONE` | website | Tình nguyện viên đã được duyệt | `campaign.service.ts > adminFinalizeCampaignCompletion()` → `notifyApprovedVolunteersCampaignDone()` | `enqueueWebsiteNotificationsToUsers()` | `campaignDone` |
| 13 | `CAMPAIGN_COMPLETION_APPROVED_BY_ADMIN` | website | Owner tổ chức | `adminFinalizeCampaignCompletion()` → `notifyOrganizationOwnerOfCompletionReview("approved")` | `enqueueWebsiteNotificationsToUsers()` | `campaignDone` |
| 14 | `CAMPAIGN_COMPLETION_REJECTED_BY_ADMIN` | website | Owner tổ chức | `campaign.service.ts > adminRejectCampaign()` (chỉ khi campaign đang `_STATUS_WAITING_CONFIRMED`) → `notifyOrganizationOwnerOfCompletionReview("rejected")` | `enqueueWebsiteNotificationsToUsers()` | `campaignCompletionRejected` |
| 15 | `REPORT_STATUS` (payload `status: "COMPLETED"`) | website | Người gửi report | `IS/src/modules/report/report.service.ts > adminMarkReportDone()` | `IS/src/modules/report/report-status-notify.client.ts > enqueueReportStatusWebsiteNotification()` | `reportStatus` |
| 16 | `REPORT_APPROVED` / `REPORT_REJECTED` | website | Người gửi report | `report.service.ts > adminVerifyReport()` → `notifyOwnerOfReportModeration()` | `enqueueReportApprovedWebsiteNotification()` / `enqueueReportRejectedWebsiteNotification()` | Có gọi filter, kind admin-only → luôn bật |
| 17 | `ORG_APPLICATION_OTP` | email (`toEmail`) | Email người nộp hồ sơ | `IS/src/modules/organization_application/organization-application-otp.service.ts > requestOtp()` | `IS/src/modules/organization_application/organization-application-notify.client.ts > enqueueApplicationOtpEmail()` | Không |
| 18 | `ORG_APPLICATION_RECEIVED` | email | Người nộp hồ sơ | `organization-application.service.ts > createApplication()` | `enqueueApplicationReceivedEmail()` | Không |
| 19 | `ORG_APPLICATION_NEEDS_INFO` | email | Người nộp hồ sơ | `organization-application-admin.service.ts > requestMoreInfo()` | `enqueueApplicationNeedsInfoEmail()` | Không |
| 20 | `ORG_APPLICATION_REJECTED` | email | Người nộp hồ sơ | `organization-application-admin.service.ts > reject()` | `enqueueApplicationRejectedEmail()` | Không |
| 21 | `ORG_ACCOUNT_ACTIVATION` | email | Email tổ chức vừa được cấp tài khoản ORG | Outbox event `ORG_ACCOUNT_PROVISION` (phát trong `organization-application-admin.service.ts > approve()`) → `IS/src/outbox/outbox-relay.bootstrap.ts` → `organization-account-provision.publisher.ts > OrganizationAccountProvisionPublisher.publish()` (chỉ khi có `activationToken` mới) | `enqueueOrgAccountActivationEmail()` | Không |

Các client của organization_application tự thêm `appName` (env `APP_NAME` của IS, mặc định `"DA2"`) và `locale` (mặc định `"vi"`) vào payload.

**Kind có trong enum nhưng không có nơi nào kích hoạt**: `RESET_PASSWORD`, `REPORT_READY`, `TASK_ASSIGNED`, `GENERIC`, `CAMPAIGN_SUBMISSION_PENDING_REVIEW`, `CAMPAIGN_SUBMISSION_APPROVED` (hai kind sau có hàm client `enqueueCampaignSubmission*Notification()` nhưng không được gọi).

## 6. Job nền, cron, worker

- **Worker SQS** `NotificationSendWorker` (job `SEND_NOTIFICATION`). Được khởi động ở **cả hai** entry: `src/worker.ts` (`npm run worker`) **và** `src/index.ts` (dòng `import "./worker"`), nên process API cũng poll SQS. Bằng chứng: `NS/src/index.ts`, `NS/src/worker.ts`.
- Vòng đời một message (`ecolink-server/shared/da2-queue/src/core/queue-worker.ts > QueueWorker.processMessage()`):
  1. Long-poll `ReceiveMessage` (batch 5, wait 20s, visibility 120s: mặc định cứng).
  2. Parse envelope; sai `jobType` → `markFailed` và xoá message.
  3. `markProcessing`: chỉ cập nhật nếu status ∈ {12 PENDING, 22 INPROCESS} → 22, `attempts = receiveCount`. Không cập nhật được (job đã COMPLETED/FAILED) → xoá message như "stale".
  4. `NotificationSendWorker.process()` kiểm tra `type ∈ {email, website}` và `kind` là string → `NotificationProcessor.process()`: kiểm tra kind hợp lệ → `getChannel(type)` → `normalizePayload` (mọi giá trị thành string, null/undefined thành `""`) → `channel.deliver()`.
  5. Thành công: xoá message, `markSucceeded` (17 COMPLETED, `processedAt`).
  6. Lỗi: nếu `receiveCount >= 5` → xoá message, `markFailed` (23). Ngược lại đổi visibility = `min(900, 30 * 2^(receiveCount-1))` giây và `markRetryScheduled` (12).
- **Kênh email** (`email.channel.ts > deliver()`): xác định người nhận (`payload.toEmail` chỉ cho phép với 6 kind ở `direct-email-kinds.ts`; ngược lại tra identity theo `userId`), `locale` = `vi` nếu `payload.locale == "vi"` ngược lại `en`, render `email.subject/text/html` → gửi SMTP → ghi `Notification` (EMAIL). Nếu `SMTP_HOST` rỗng: không gửi, vẫn ghi DB, log `smtp_not_configured`.
- **Kênh in-app** (`in-app.channel.ts > deliver()`): bắt buộc `userId`, render `website.title/body` cho cả `en` và `vi`, lưu `title/body` bản `en`, và `payload.locales = { en: {title, body}, vi: {title, body} }`.
- **Template engine** (`NS/src/modules/templates/notification-template.engine.ts`): tìm file theo thứ tự `.vi.hbs` → `.en.hbs` → `.hbs` (locale vi) hoặc `.en.hbs` → `.hbs` (locale en); thiếu file → throw (job sẽ retry rồi FAILED). Với kênh website, biến `fooEn`/`fooVi` trong payload được dùng để điền `foo` theo locale (`payloadForLocale()`); kênh email **không** áp dụng cơ chế này. Template compile được cache trong bộ nhớ. Thư mục template tính theo `process.cwd()/templates/notifications`.
- **Không có cron** nào trong service.

Ma trận template hiện có (thư mục `NS/templates/notifications/`):

| Kind | website | email |
|---|---|---|
| CAMPAIGN_CREATED, CAMPAIGN_DONE, CAMPAIGN_COMPLETION_PENDING_ADMIN, CAMPAIGN_COMPLETION_REJECTED_BY_ADMIN, REPORT_STATUS, REPORT_READY, TASK_ASSIGNED, VOLUNTEER_REQUEST, VOLUNTEER_APPROVED, VOLUNTEER_REJECTED, RESET_PASSWORD, GENERIC | có | có |
| CAMPAIGN_COMPLETION_APPROVED_BY_ADMIN, CAMPAIGN_VERIFY_INVITE, CAMPAIGN_COMPLETION_VERIFY_INVITE, ORGANIZATION_APPROVED, ORGANIZATION_REJECTED, REPORT_APPROVED, REPORT_REJECTED | có | **không** |
| ORGANIZATION_CONTACT_VERIFY, ORG_APPLICATION_OTP, ORG_APPLICATION_RECEIVED, ORG_APPLICATION_NEEDS_INFO, ORG_APPLICATION_REJECTED, ORG_ACCOUNT_ACTIVATION | **không** | có |
| CAMPAIGN_SUBMISSION_PENDING_REVIEW, CAMPAIGN_SUBMISSION_APPROVED | **không** | **không** |

## 7. Phụ thuộc vào service khác và dịch vụ bên ngoài

| Phụ thuộc | Mục đích | Bằng chứng |
|---|---|---|
| PostgreSQL (`notificationdb`) qua Prisma | Bảng `notifications`, `notification_jobs` | `NS/prisma/schema.prisma`, `NS/src/lib/prisma.ts` |
| AWS SQS (LocalStack khi local) qua `@da2/queue` | Hàng đợi `SEND_NOTIFICATION` | `NS/src/queue/notification-sqs-queue-factory.ts` |
| identity-service | Tra email theo userId | `NS/src/lib/identity-user.client.ts` |
| SMTP (nodemailer; ví dụ trong `.env.example`: Gmail, Mailtrap, Mailpit) | Gửi email | `NS/src/channels/email/email.provider.ts` |
| Datadog APM (`dd-trace`) | Tracing | `NS/src/tracer.ts` |
| `@da2/constants`, `@da2/express-swagger` | HTTP status, GlobalStatus, Swagger | `NS/package.json` |

## 8. Biến môi trường

| Biến | Ý nghĩa | Đọc ở |
|---|---|---|
| `PORT` | Cổng HTTP (mặc định 3003) | `src/index.ts` |
| `NODE_ENV` | `development` bật log query Prisma và trả `stack` khi lỗi 500 | `src/lib/prisma.ts`, `src/middleware/error.middleware.ts` |
| `DATABASE_URL` | Kết nối Postgres | `prisma/schema.prisma` |
| `JWT_SECRET` | Verify JWT (bắt buộc, phải trùng identity) | `src/utils/jwt.utils.ts` |
| `JWT_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` | Có trong `.env.example`, **không được code NS đọc** | — |
| `IDENTITY_SERVICE_URL`, `INTERNAL_IDENTITY_API_KEY` | Gọi identity lấy email | `src/lib/identity-user.client.ts` |
| `INTERNAL_NOTIFICATION_API_KEY` | API key cho `POST /jobs` | `src/middleware/internal-auth.middleware.ts` |
| `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SQS_ENDPOINT` (hoặc `AWS_ENDPOINT_URL`) | Cấu hình SQS client | `src/queue/notification-sqs-queue-factory.ts` |
| `SQS_NOTIFICATION_QUEUE_URL` | URL queue (bắt buộc; thiếu → throw khi khởi tạo QueueRunner) | `src/queue/register.ts` |
| `NOTIFICATION_SEND_CONCURRENCY` | Truyền vào `concurrency` của route; **`QueueRunner` không dùng giá trị này** (luôn 1 worker/route) | `src/queue/register.ts` |
| `WORKER_SQS_WAIT_TIME_SECONDS`, `WORKER_SQS_VISIBILITY_TIMEOUT_SECONDS`, `WORKER_BATCH_SIZE`, `WORKER_MAX_RECEIVE_COUNT`, `WORKER_RETRY_BASE_SECONDS`, `WORKER_MAX_RETRY_DELAY_SECONDS` | Có trong `.env.example` (comment), **không được code đọc**; threshold truyền `{}` nên luôn dùng mặc định | — |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` | SMTP; `SMTP_HOST` rỗng → không gửi mail. `SMTP_FROM` fallback `SMTP_USER` rồi `noreply@localhost`. Transporter cache, đổi cần restart | `src/channels/email/email.provider.ts` |
| `CORS_ORIGIN` | Origin CORS | `src/index.ts` |
| `SWAGGER_SERVER_URL` | Server URL trong OpenAPI | `src/index.ts` |
| `DD_ENV`, `DD_VERSION` (+ `DD_AGENT_HOST` ở K8s) | Datadog | `src/tracer.ts` |

## 9. Vấn đề cần xác nhận / [CHƯA HOÀN THIỆN]

1. **Worker chạy hai lần**: `src/index.ts` có `import "./worker"`, trong khi còn script `worker`/`start:worker:production` riêng. Khi deploy cả API và worker, sẽ có 2+ consumer (không sai về logic nhờ `markProcessing`, nhưng có thể không phải chủ ý). Ngoài ra `worker.ts` đăng ký SIGINT/SIGTERM gọi `process.exit(0)` ngay cả trong process API.
2. **Gửi email trùng khi retry**: kênh email gửi SMTP **trước** rồi mới `prisma.notification.create`. Nếu ghi DB lỗi → job retry → email gửi lại. `markProcessing` cho phép status 22 nên message được nhận lại vẫn xử lý lại (at-least-once).
3. **Kind thiếu template** → job luôn thất bại sau 5 lần: `CAMPAIGN_SUBMISSION_PENDING_REVIEW`, `CAMPAIGN_SUBMISSION_APPROVED` (cả 2 kênh); kênh email cho các kind chỉ có template website (ví dụ `REPORT_APPROVED`); kênh website cho các kind chỉ có email (ví dụ `ORG_APPLICATION_OTP`). Controller không kiểm tra trước.
4. **Kind không có trigger** [CHƯA HOÀN THIỆN]: `RESET_PASSWORD` (identity-service `auth.controller.ts > requestPasswordReset` trả thẳng `resetToken` trong response thay vì gửi email: nên xác nhận với identity), `REPORT_READY`, `TASK_ASSIGNED`, `GENERIC`, hai kind `CAMPAIGN_SUBMISSION_*`. Nhiều hàm client ở `IS/src/modules/campaign/notification-jobs.client.ts` là code chết: `enqueueCampaignCreatedWebsiteNotification`, `enqueueCampaignDoneWebsiteNotification`, `enqueueCampaignCompletionRejectedByAdminWebsiteNotification`, `enqueueCampaignCompletionApprovedByAdminWebsiteNotification`, `enqueueCampaignVerifyInviteNotification`, `enqueueCampaignCompletionVerifyInviteNotification`, `enqueueCampaignSubmission*`.
5. **`POST /jobs` lộ qua gateway** (`/api/v1/notifications/*` proxy toàn bộ). Chỉ còn API key bảo vệ; so sánh bằng `!==` (không constant-time). Ai có key có thể gửi email tới bất kỳ địa chỉ nào qua 6 kind cho phép `toEmail`.
6. **Validation `type` lỏng**: controller chỉ kiểm `isString().notEmpty()`. Giá trị như `"sms"` được nhận (202) rồi worker ném "Invalid notification payload" → retry 5 lần mới FAILED. Tương tự `in-app`/`inapp` bị worker từ chối dù `InAppNotificationChannel.supports()` nhận được. Controller chỉ bắt buộc `userId` khi `type === "website"` (không so sánh không phân biệt hoa thường).
7. **`payload` không kiểm kiểu giá trị**: chỉ `isObject()`; object lồng nhau bị `String(v)` thành `"[object Object]"`. Không giới hạn kích thước.
8. **Không kiểm tra `userId` tồn tại** khi tạo thông báo website.
9. **Rò rỉ dữ liệu trong `payload` DB**: thông báo EMAIL lưu nguyên payload, gồm `otp`, `activationUrl` (token kích hoạt), `verifyUrl`. `.env.example` còn nói rõ OTP "readable from the `notifications` table". Cân nhắc bảo mật.
10. **Không có retention / dọn dẹp** `notifications` và `notification_jobs`.
11. **Job FAILED khi gửi SQS lỗi không bao giờ được gửi lại** (không có relay quét outbox). Bên gọi (incident) nhận 400 và phần lớn chỉ log (best-effort).
12. **Env khai báo nhưng không dùng**: `WORKER_*`, `NOTIFICATION_SEND_CONCURRENCY` (không có tác dụng), `JWT_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN`.
13. **Preferences fail-open**: nếu identity lỗi, incident gửi cho tất cả user. `CAMPAIGN_COMPLETION_PENDING_ADMIN` bỏ qua hoàn toàn bước lọc. Kind `ORGANIZATION_*`/`REPORT_APPROVED|REJECTED` vẫn gọi filter dù luôn được bật (gọi thừa).
14. **Không có "đánh dấu tất cả đã đọc"**; `markRead` luôn ghi đè `readAt` bằng thời điểm mới dù đã đọc trước đó. Có race nhỏ giữa `findFirst` và `update` (không ảnh hưởng quyền vì `update` theo `id` chỉ sau khi đã check chủ sở hữu).
15. **`.env.example` có một giá trị `SMTP_USER` trông như địa chỉ thật (bị cắt)**: nên thay bằng placeholder.
16. **Mô tả sai trong tên kind**: template VOLUNTEER_* dùng biến `reportTitle` để chứa tên chiến dịch/tổ chức (comment trong `IS/.../notification-jobs.client.ts`).
