# reward-service

> Tài liệu viết hoàn toàn dựa trên code tại `ecolink-server/services/reward-service` (commit hiện tại). Đường dẫn bằng chứng tính từ `/Users/ngoc/ecolink`.
> Cơ chế dịch tự động (TRANSLATE_TEXT) được mô tả chi tiết riêng ở [translation-worker.md](./translation-worker.md).

## 1. Trách nhiệm của service

reward-service là nơi lưu trữ và tính toán mọi thứ liên quan đến điểm thưởng:

| Mảng | Nội dung | Bằng chứng |
|---|---|---|
| Difficulty (**deprecated**) | Bậc độ khó chiến dịch cũ (`maxVolunteers`, `suggestedMinVolunteers`, `greenPoints`). incident-service và client không còn đọc từ khi bỏ độ khó (spec 013); bảng, `/api/v1/difficulties`, `/internal/v1/difficulties*` còn lại tới đợt dọn dẹp (đợt C); `campaign-reward-estimate` đã gỡ (spec 014) | `.../src/modules/difficulty/difficulty.service.ts` |
| Quy tắc điểm | Mức điểm admin đặt (`gamification_point_rules`: hoàn thành chiến dịch, điểm cơ sở và mốc upvote report, report hoàn thành, **số ngày hết hạn của điểm** `point_expiration_days`) kèm lịch sử thay đổi (`gamification_point_rule_history`); mọi strategy cộng điểm đọc quy tắc đang hiệu lực lúc cộng | `.../src/modules/gamification/gamification-config.service.ts` |
| Điểm xanh (spec 014) | Một loại điểm duy nhất. Sổ cái `green_point_transactions` (khoá idempotent theo user) là sổ duy nhất; mỗi lần cộng tạo 1 dòng sổ + 1 **lô** (`green_point_lots`) có hạn dùng. Số dư = tổng `remaining` của các lô còn hạn. Job nền ghi `EXPIRE` cho lô hết hạn | `.../src/modules/green-point/green-point-ledger.util.ts > insertLedgerOnce(), creditPoints()`, `green-point-lot.util.ts`, `green-point-expiry.job.ts` |
| Gift / đổi quà | Danh mục quà, đổi quà theo giá niêm yết (trừ lô sắp hết hạn trước, ghi lại lô đã trừ), đơn đổi quà với trạng thái vận chuyển, hoàn điểm về đúng lô khi huỷ | `.../src/modules/gift/gift.service.ts` |
| Nguồn quà và kho (spec 015) | Nguồn quà (nhà tài trợ, Ecolink tự mua, quyên góp), nhập kho theo lô, ghi giảm có lý do, lịch sử kho chỉ thêm, lô quà hết hạn, báo cáo theo nguồn (JSON/CSV). Tồn kho = tổng lô còn hạn, `gifts.stock_remaining` là cache | `.../src/modules/gift/gift-source.*`, `gift-stock.*`, `gift-stock-expiry.job.ts`, `gift-source-report.ts` |
| Facebook recognition | Nhận sự kiện hoàn thành chiến dịch, sinh caption bằng ai-service, đăng lên Facebook Graph và/hoặc webhook | `.../src/modules/facebook-recognition/facebook-recognition.service.ts` |
| Dịch tự động | Dịch tên/mô tả Gift và tên Difficulty (deprecated) sang vi/en bất đồng bộ qua SQS | `.../src/queue/workers/translation.worker.ts` |

reward-service KHÔNG tự phát sự kiện sang service khác. Nó tiêu thụ hàng đợi `reward-intake` do incident-service đẩy vào (outbox) và gọi HTTP đồng bộ sang identity-service, incident-service, ai-service, Facebook Graph.

## 2. Cấu trúc thư mục / module chính

```
reward-service/
├── prisma/schema.prisma, migrations/, seed.ts
└── src/
    ├── index.ts        # Express app, mount routes, import "./worker" => API process cũng chạy worker
    ├── worker.ts       # startAllQueues(), startGreenPointExpiryJob(), xử lý SIGINT/SIGTERM
    ├── scripts/reconcile-green-points.ts    # đối soát sổ ↔ lô khi triển khai spec 014 (chạy tay)
    ├── tracer.ts       # dd-trace (Datadog)
    ├── config/prisma.client.ts
    ├── constants/http-status.ts             # re-export @da2/constants
    ├── middleware/     # auth, require-admin, internal-reward-auth, error
    ├── internal/internal.routes.ts          # /internal/v1/*
    ├── utils/          # identity-user.client.ts, incident-resource.client.ts, jwt.utils.ts
    ├── openapi/route-models.ts              # model cho swagger
    ├── queue/          # bootstrap (dispatcher + 4 SQS queue), register (khởi động worker), job stores, workers/
    └── modules/
        ├── difficulty/  gift/  user-points/
        ├── green-point/ (service, factory, 3 strategy, ledger util, lot util, expiry job, constants)
        ├── gamification/ (controller, routes, config — chỉ còn quy tắc điểm)
        ├── facebook-recognition/
        └── translation/ (translation.client.ts, translation.types.ts)
```

Process: `npm run start` (`dist/index.js`) chạy cả HTTP server lẫn toàn bộ worker, vì `src/index.ts` có `import "./worker"`. `npm run worker` (`dist/worker.js`) chỉ chạy worker. Dockerfile dùng `start:production` (migrate deploy + start) (`ecolink-server/services/reward-service/Dockerfile`, `package.json > scripts`).

## 3. Cơ chế xác thực & middleware

| Middleware | Hành vi | Bằng chứng |
|---|---|---|
| `authenticate` | Lấy token từ header `Authorization: Bearer <token>`, nếu không có thì từ cookie `accessToken`. Thiếu token → 401 `TOKEN_MISSING` ("Authentication token is required"). Verify lỗi → 401 `TOKEN_INVALID` ("Invalid token"). Gán `req.user = { userId, email, role? }` | `.../src/middleware/auth.middleware.ts > authenticate()` |
| `tryAuthenticate` | Như trên nhưng không bao giờ chặn request; token sai/thiếu thì bỏ qua | `.../src/middleware/auth.middleware.ts > tryAuthenticate()` |
| `verifyToken` | `jwt.verify(token, JWT_SECRET)`; nếu thiếu env dùng fallback cứng `"fallback-secret-key"` | `.../src/utils/jwt.utils.ts > verifyToken()` |
| `requireAdmin` | `req.user.role.toLowerCase() === "admin"`, nếu không → 403 `FORBIDDEN` "Admin access required" | `.../src/middleware/require-admin.middleware.ts > requireAdmin()` |
| `requireInternalRewardApiKey` | Áp cho mọi route `/internal/v1`. Env `INTERNAL_REWARD_API_KEY` không có → 500 "INTERNAL_REWARD_API_KEY is not configured"; header `x-internal-api-key` sai/thiếu → 401 "Invalid internal API key" | `.../src/middleware/internal-reward-auth.middleware.ts` |
| `errorHandler` | Lỗi không bắt → 500 `INTERNAL_SERVER_ERROR` | `.../src/middleware/error.middleware.ts` |

Envelope phản hồi chuẩn (từ `@da2/constants`): thành công `{ success: true, code, message, data }`, lỗi `{ success: false, code, message, ...extra }`. Lỗi validation của express-validator trả 400 `VALIDATION_ERROR` kèm `errors: [...]` (`ecolink-server/shared/da2-constants/src/http-status.ts > sendError(), sendSuccess()`).

Chỉ có 2 vai trò được phân biệt trong service: `admin` và "người dùng đã đăng nhập bất kỳ". Không có kiểm tra quyền theo tổ chức.

## 4. Danh sách API endpoint

Tổng: **20 endpoint** = 16 endpoint dưới `/api/v1` (đều được api-gateway proxy) + 3 endpoint nội bộ `/internal/v1` (gateway KHÔNG proxy, chỉ incident-service gọi trực tiếp) + `GET /health` (không qua gateway).

Spec 014 (2026-10-21) đã gỡ hẳn leaderboard, season, badge, gamification summary, metric metadata và các cấu hình admin `sp-rules`, `multipliers`, `season-schedules`, `payout-tiers`, `badges`: các path `/leaderboard`, `/leaderboard/me`, `/seasons/*`, `/me/gamification/*`, `/me/badges`, `/gamification/campaign-reward-estimate`, `/gamification/leaderboards/:metric[/me]`, `/admin/gamification/{sp-rules,multipliers,season-schedules,payout-tiers,badges}`, `/admin/seasons*`, `/metric-tables`, `/metric-columns` giờ trả 404 (gateway cũng không proxy nữa). Bảng dữ liệu cũ vẫn giữ (mục 9, ý 24).

Tất cả route `/api/v1/*` được mount trong `.../src/index.ts`. Path dưới đây là path đầy đủ qua gateway (gateway giữ nguyên path).

### 4.1 Difficulty (2) — deprecated

Không còn service nào trong hệ thống gọi (incident-service và client đã bỏ độ khó, spec 013); bỏ ở đợt dọn dẹp (đợt C).

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/difficulties` | Không | Public | query `page` int ≥1 (mặc định 1), `limit` int 1..100 (mặc định 20) | `{ difficulties: DifficultyResponse[], page, limit, total, totalPages }`. Chỉ bản ghi `deletedAt = null`, sắp theo `level asc`. Trường `name` luôn `null`; `nameVi = nameVi ?? name` | 400 VALIDATION_ERROR, 500 | `.../modules/difficulty/difficulty.api.routes.ts > GET /difficulties` → `difficulty.service.ts > listActive()` |
| PUT | `/api/v1/difficulties/:id` | Bearer | admin | param `id` UUID; body tuỳ chọn: `name`, `nameVi`, `nameEn` (trim, 1..64), `maxVolunteers` (null hoặc int ≥1), `suggestedMinVolunteers` (null hoặc int ≥1), `greenPoints` (int ≥0) | `{ difficulty }` | 400, 401, 403, 404 (không tìm thấy hoặc đã soft delete), 500 | `difficulty.api.routes.ts > PUT /difficulties/:id` → `difficulty.service.ts > updateById()` |

`updateById()` điền sẵn ngôn ngữ còn thiếu bằng text nguồn và enqueue job dịch (xem translation-worker.md). Không có endpoint tạo/xoá difficulty; dữ liệu đến từ seed.

### 4.2 Gift và đơn đổi quà (7)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/gifts` | Tuỳ chọn (`tryAuthenticate`) | Public; admin được lọc `isActive` | query: `page`≥1, `limit` 1..100, `search` (so khớp không phân biệt hoa thường trên `name`/`nameVi`/`nameEn`), `inStock` "true"/"false" (chỉ "true" có tác dụng: cache `stockRemaining` > 0), `isActive` "true"/"false" (chỉ admin), `greenPointsMin`/`greenPointsMax` int ≥0, `sortBy` ∈ createdAt/name/greenPoints (mặc định createdAt desc), `sortOrder` asc/desc | `{ gifts: GiftResponse[], page, limit, total, totalPages }`; `name` và `description` luôn `null`, kèm `media {id,url,type}`, `stockRemaining` (tổng `remaining` các lô còn hạn, luôn là số), `sponsors[] {id,name,logoUrl}` (BR-238), `expiringSoon {quantity, earliestAt}` chỉ cho admin, người khác `null` | 400 (validation; hoặc "greenPointsMin must be <= greenPointsMax"), 500 | `.../modules/gift/gift.api.routes.ts > GET /gifts` → `gift.service.ts > listGifts()` |
| GET | `/api/v1/gifts/:id` | Tuỳ chọn (`tryAuthenticate`) | Public; admin nhận `expiringSoon` | `id` UUID | `{ gift }` (không lọc `isActive`, chỉ lọc `deletedAt null`) | 400, 404 "Gift not found", 500 | `gift.api.routes.ts > GET /gifts/:id` → `getGiftById()` |
| POST | `/api/v1/gifts` | Bearer | admin | body: `name` bắt buộc (1..255), `nameVi`/`nameEn` tuỳ chọn (1..255), `imageUrl` tuỳ chọn isURL, `description` bắt buộc string, `descriptionVi`/`descriptionEn` tuỳ chọn, `greenPoints` bắt buộc int ≥0, `isActive` boolean (mặc định true). `stockRemaining` trong body bị bỏ qua (spec 015); quà mới tồn kho 0 | 201 `{ gift }` | 400, 401, 403, 500 | `POST /gifts` → `gift.service.ts > create()` |
| PUT | `/api/v1/gifts/:id` | Bearer | admin | như POST nhưng mọi field tuỳ chọn; `imageUrl` rỗng → gỡ media | `{ gift }` | 400, 401, 403, 404, 500 | `PUT /gifts/:id` → `updateById()` |
| POST | `/api/v1/gifts/:id/redeem` | Bearer | Mọi user | `id` UUID; body: `phoneNumber` là số di động Việt Nam: bỏ khoảng trắng và `.-()` rồi khớp `^(0\d{9}|\+84\d{9})$`, lưu dạng chỉ có chữ số; `recipientName` 1..100; `pickupLocation` 1..1000 (địa chỉ chọn trên bản đồ Goong); `deliveryLatitude` −90..90 và `deliveryLongitude` −180..180 bắt buộc (ghim); `addressDetail` ≤ 255 và `deliveryNote` ≤ 500 không bắt buộc (rỗng lưu null) | `{ redemption: GiftRedemptionResponse }` (HTTP 200) kèm `code`, `carrier`, `trackingCode`, `shippedAt`, `recipientName`, `addressDetail`, `deliveryLatitude`, `deliveryLongitude`, `deliveryNote`, `cancelReason`, `restocked`, `allowedTransitions` | 400 (validation / "Missing id or user"), 401, 404 "Gift not found" (không tồn tại, đã xoá hoặc `isActive=false`), 422 "Gift is out of stock", 422 "Insufficient green points", 409 "Please retry" (vẫn xung đột Serializable sau 15 lần chạy lại), 500 | `gift.api.routes.ts > handleGiftRedeem()` → `gift.service.ts > redeem()` |
| GET | `/api/v1/admin/gift-redemptions` | Bearer | admin | query `page`, `limit` 1..100, `code?` (mã đơn, mọi cách viết; sai định dạng → danh sách rỗng), `status` ∈ PROCESSING/SHIPPED/DELIVERED/CANCELLED, `sortBy` ∈ createdAt/greenPointsSpent/statusUpdatedAt (mặc định createdAt), `sortOrder` (mặc định desc) | `{ redemptions: AdminGiftRedemptionListItemResponse[] (kèm snapshot gift và user {id,name,avatar} lấy từ identity, `source`, `cancelReason`, `restocked`, `code`, `carrier`, `trackingCode`, `shippedAt`, `allowedTransitions` theo vai trò ADMIN), page, limit, total, totalPages }` | 400, 401, 403, 500 | `GET /admin/gift-redemptions` → `listRedemptionsForAdmin()` |
| GET | `/api/v1/admin/gift-redemptions/:code` | Bearer | admin | `code` mã đơn (chuẩn hoá bằng `order-code.util.ts > normalizeOrderCode()`) | `{ redemption: AdminGiftRedemptionDetailResponse }`: như item admin + `events[]` mới nhất trước, mỗi event có `actor {id, name}` (tên lấy qua `fetchUsersByIds`; mục dựng lại `actor` null) | 401, 403, 404 "Redemption not found" | `getAdminRedemptionByCode()` |
| PATCH | `/api/v1/admin/gift-redemptions/:id/status` | Bearer | admin (`requireAdmin`, spec 015) | `id` UUID; body `status` ∈ 4 giá trị enum, `cancelReason?` string (bắt buộc 1..500 sau trim khi huỷ), `restock?` boolean (bắt buộc khi huỷ đơn SHIPPED), `carrier?` + `trackingCode?` (bắt buộc khi chuyển SHIPPED, BR-230) | `{ redemption }` | 400 (field `cancelReason` / `restock` / `carrier` / `trackingCode`), 401, 404 "Redemption not found", 409 "Order was already shipped by someone else" (đơn đã SHIPPED mà gửi hãng / mã khác giá trị đang lưu), 409 "Please retry", 422 "Cannot change redemption status from X to Y", 500 | `PATCH /admin/gift-redemptions/:id/status` → `updateRedemptionStatus()` → `changeStatus()`; chuyển sang SHIPPED thành công thì sau commit gửi thông báo `GIFT_ORDER_SHIPPED` (lỗi chỉ log) |
| PATCH | `/api/v1/admin/gift-redemptions/:id/tracking` | Bearer | admin | `id` UUID; body `carrier` 1..100, `trackingCode` 1..100 khớp `^[A-Za-z0-9._/-]+$` (sau trim) | `{ redemption }` | 400, 401, 403, 404, 409 "Please retry", 422 "Tracking can only be changed while the order is SHIPPED" | `updateTracking()`: cả hai giá trị không đổi thì không ghi gì; đổi thì ghi event `TRACKING_UPDATED` kèm giá trị cũ |

Đổi quà (`gift.service.ts > redeem()`, transaction `Serializable`): tìm quà active → giá = `gift.greenPoints` (không giảm giá, BR-226) → số dư lô còn hạn < giá thì 422 → tạo đơn PROCESSING → `green-point-lot.util.ts > spendLotsFifo()` trừ lô theo `expires_at` rồi `created_at` tăng dần, mỗi lần trừ là `updateMany` có điều kiện `remaining >= take` và ghi một dòng `green_point_lot_spends` (BR-262) → ghi sổ `GIFT_REDEEM` −giá (khoá `gift_redeem:<redemptionId>`). Kho (spec 015): ngay sau khi tạo đơn, `gift-stock.util.ts > takeOneFromBatches()` lấy 1 món từ lô còn hạn (hạn sớm trước, lô không hạn sau cùng, rồi `received_at`), trừ có điều kiện `remaining > 0`, ghi `batch_id` vào đơn, movement `REDEEM_OUT` −1 (actor = người đổi) và cache −1; hết lô → 422 "Gift is out of stock", rollback cả đơn. Xung đột tuần tự hoá (P2034, hoặc P2010 với 40001/40P01, `isSerializationConflict()`) → chạy lại cả transaction tối đa 15 lần với backoff full-jitter có trần 250 ms; hết lượt → 409 "Please retry". Alias `POST /gifts/:id/exchange` đã gỡ (spec 016).

Huỷ đơn có `greenPointsSpent > 0` (`updateRedemptionStatus()` → `refundRedemptionPoints()`): ghi sổ `GIFT_REDEEM_REFUND` +đã tiêu (khoá `gift_redeem_refund:<redemptionId>`; đã có thì dừng). Có `green_point_lot_spends`: phần của lô còn hạn cộng lại vào đúng lô (giữ hạn cũ); phần của lô đã hết hạn ghi ngay `EXPIRE` −phần đó (resourceType `GREEN_POINT_LOT`, khoá `expire:<lotId>:<redemptionId>`), không cộng lại (BR-263). Không có lot spend (đơn tạo trước spec 014) → tạo một lô mới gắn với dòng hoàn, hạn đủ `pointExpirationDays` tính từ lúc huỷ (BR-264). Huỷ (spec 016, BR-229): dòng đơn bị khoá `FOR UPDATE`; cùng trạng thái → trả đơn như cũ; cần `cancelReason`; đơn SHIPPED cần `restock`; đơn PROCESSING luôn trả kho; lưu `restocked = restock && batchId != null`; trả kho qua `gift-stock.util.ts > returnToBatch()` (BR-234). Bảng chuyển trạng thái `gift.dto.ts > REDEMPTION_STATUS_TRANSITIONS` (admin) và `USER_REDEMPTION_TRANSITIONS` (chủ đơn: PROCESSING → CANCELLED, SHIPPED → DELIVERED), trả ra qua `allowedTransitionsFor(status, role)`.

Spec 017: mọi đổi trạng thái (admin lẫn chủ đơn) đi qua `gift.service.ts > changeStatus(target, body, actor)`: khoá dòng `FOR UPDATE` (theo `id`, hoặc theo `code` + `user_id` với chủ đơn), kiểm bảng chuyển theo vai trò, gửi lại đúng trạng thái hiện tại thì không đổi gì và không ghi lịch sử (trừ SHIPPED với hãng / mã khác → 409), SHIPPED cần `carrier` + `trackingCode` (`parseTracking()`, `fieldError`), rồi ghi event lịch sử trong cùng transaction. Trả `{ redemption, changed }`. `redeem()`, `changeStatus()` và `updateTracking()` cùng dùng `retryOnConflict()`: xung đột tuần tự hoá chạy lại tối đa 15 lần (full jitter, trần 250 ms), hết lượt → 409 "Please retry". `redeem()` sinh mã đơn mới mỗi lần thử (`order-code.util.ts > generateOrderCode()`), trùng mã (P2002 trên `code`, `isOrderCodeClash()`) cũng chạy lại; ghi event `CREATED`.

### 4.2a Nguồn quà và kho (10)

Mọi route: `authenticate` + `requireAdmin`, validate bằng express-validator (400 `VALIDATION_ERROR`). Response có phân trang trả `page, limit, total, totalPages`.

| Method | Path | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|
| GET | `/api/v1/admin/gift-sources` | query `page`, `limit` 1..100, `type` ∈ SPONSOR/ECOLINK/DONATION, `status` ∈ ACTIVE/INACTIVE, `search` (tên, không phân biệt hoa thường) | `{ sources[] }` (mỗi nguồn kèm `logo`, `batchCount`) | 400 | `gift-source.api.routes.ts` → `gift-source.service.ts > list()` |
| POST | `/api/v1/admin/gift-sources` | `name` 1..255, `type`, `contactName?` ≤255, `contactPhone?` 7..32 regex SĐT, `contactEmail?` email, `logoUrl?` URL, `showOnGift?`, `note?` ≤2000 | 201 `{ source }` | 400 | `create()` |
| GET | `/api/v1/admin/gift-sources/:id` | — | `{ source }` | 404 | `getById()` |
| PUT | `/api/v1/admin/gift-sources/:id` | như POST nhưng tuỳ chọn; `status`; `logoUrl: ""` gỡ logo | `{ source }` | 400, 404 | `update()` |
| GET | `/api/v1/admin/gift-sources/:id/batches` | `page`, `limit` | `{ batches[] }` (kèm `gift {id,name}`, `state`), mới nhập trước | 404 | `listBatches()` |
| GET | `/api/v1/admin/gift-sources/:id/report` | `from`, `to` (YYYY-MM-DD, +07:00), `format` json/csv | `{ report: {sourceId, from, to, totals, gifts[]} }` hoặc file CSV (BOM) | 400, 404 | `gift-source-report.ts > loadSourceReport(), toCsv()` |
| GET | `/api/v1/admin/gifts/:id/stock-batches` | `page`, `limit`, `sourceId?` | `{ batches[] }` (kèm `source`, `state`), theo thứ tự lấy hàng | 404 | `gift-stock.service.ts > listBatches()` |
| POST | `/api/v1/admin/gifts/:id/stock-batches` | `sourceId`, `quantity` ≥1, `unitValueVnd` ≥0, `receivedAt` ISO không ở tương lai, `expiresAt?` sau `receivedAt`, `reference?` ≤255, `attachmentUrls?` (≤5 URL http(s)); phải có `reference` hoặc ít nhất 1 file, `note?` | 201 `{ batch, stockRemaining }` | 400, 404, 422 "Gift source is inactive" | `stockIn()` |
| POST | `/api/v1/admin/stock-batches/:id/adjustments` | `quantity` ≥1, `reason` ∈ DAMAGED/LOST/COUNT_SHORTAGE/RETURN_TO_SOURCE, `note?` | 201 `{ batch, movement, stockRemaining }` | 400, 404, 422 "Not enough stock in batch" | `adjust()` → `gift-stock.util.ts > adjustBatch()` |
| GET | `/api/v1/admin/gifts/:id/stock-movements` | `page`, `limit`, `batchId?`, `type?` | `{ movements[] }` mới nhất trước, kèm `sourceName`, `actor {id,name}` (null = hệ thống) | 404 | `listMovements()` |

### 4.3 Điểm của tôi, lịch sử đổi quà (3)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/me/points` | Bearer | Mọi user | — | `{ balance, earnedTotal, expiringSoon: { amount, earliestAt }, spendablePoints, greenPoints, greenPointsEarnedTotal }`. `balance` = tổng `remaining` của lô còn hạn (BR-261); `earnedTotal` = tổng điểm các loại kiếm được (BR-266); `expiringSoon.amount` = điểm trong lô hết hạn trong 7 ngày tới, `earliestAt` = ngày hết hạn sớm nhất của các lô đó (ISO) hoặc null (BR-265). Field cũ giữ một bản phát hành: `spendablePoints` = `greenPoints` = `balance`, `greenPointsEarnedTotal` = `earnedTotal` | 401, 500 | `.../modules/user-points/user-points.api.routes.ts > GET /me/points` → `user-points.service.ts > getPoints()` |
| GET | `/api/v1/me/points/transactions` | Bearer | Mọi user | query `page`, `limit` 1..100, `type` (string: "earned" → points>0, "spent" → points<0, giá trị khác → so khớp đúng `type`), `sortBy` ∈ createdAt/points/type, `sortOrder` | `{ transactions: (GreenPointTransaction + resource)[], page, limit, total, totalPages }`. `resource` được làm giàu: CAMPAIGN/REPORT (dòng mốc vote có `resourceType = REPORT` từ spec 014) gọi incident-service `/api/v1/campaigns/by-ids`, `/api/v1/reports/by-ids` (chuyển tiếp header Authorization), USER gọi identity-service (trừ dòng `MIGRATION_ADJUST`), GIFT_REDEMPTION đọc DB cục bộ; `GREEN_POINT_LOT` (dòng `EXPIRE`) và `MIGRATION_ADJUST` có `resource = null`. `type` có thêm `EXPIRE`, `MIGRATION_ADJUST` | 400, 401, 500 | `GET /me/points/transactions` → `getTransactions()` |
| GET | `/api/v1/me/redemptions` | Bearer | Mọi user | query `page`, `limit`, `code?` (mã đơn, mọi cách viết), `sortBy` ∈ createdAt/greenPointsSpent/statusUpdatedAt, `sortOrder` | `{ redemptions: GiftRedemptionListItemResponse[], page, limit, total, totalPages }`; mỗi đơn kèm `code`, `carrier`, `trackingCode`, `shippedAt`, `cancelReason`, `allowedTransitions` theo vai trò USER | 400, 401, 500 | `GET /me/redemptions` → `gift.service.ts > listRedemptionsForUser()` |
| GET | `/api/v1/me/redemptions/:code` | Bearer | Chủ đơn | `code` mã đơn | `{ redemption: GiftRedemptionDetailResponse }` (item + `events[]`, **không** có `actor`) | 401, 404 "Redemption not found" (không có, không phải của mình, hoặc mã sai định dạng) | `user-points.api.routes.ts` → `getMyRedemptionByCode()` |
| POST | `/api/v1/me/redemptions/:code/cancel` | Bearer | Chủ đơn | body `cancelReason` 1..500 sau trim | `{ redemption: GiftRedemptionDetailResponse }` | 400 field `cancelReason`, 401, 404, 409 "Please retry", 422 (đơn không còn PROCESSING) | `cancelMyRedemption()` → `changeStatus()` vai trò USER: hoàn điểm và trả kho như admin huỷ đơn PROCESSING |
| POST | `/api/v1/me/redemptions/:code/confirm-delivery` | Bearer | Chủ đơn | — | `{ redemption: GiftRedemptionDetailResponse }` | 401, 404, 409 "Please retry", 422 (đơn chưa SHIPPED hoặc đã huỷ) | `confirmMyRedemption()` → `changeStatus()` SHIPPED → DELIVERED |

### 4.4 Admin quy tắc điểm (3)

Mọi route: `authenticate` + `requireAdmin` (401/403). Body KHÔNG dùng express-validator; kiểm tra thủ công trong controller.

| Method | Path | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|
| GET | `/api/v1/admin/gamification/point-rules` | — | `{ rules: { id, campaignCompletionPoints, baseReportPoint, reportMilestoneThresholds, reportCompletionPoints, pointExpirationDays, effectiveFrom, updatedAt } }` (bản ghi active mới nhất theo `effectiveFrom`) | 404 "No active point rules", 500 | `adminGetPointRules()` |
| PATCH | `/api/v1/admin/gamification/point-rules` | Đủ 5 field (ngữ nghĩa PUT): `campaignCompletionPoints`, `baseReportPoint`, `reportCompletionPoints` int ≥0; `reportMilestoneThresholds` mảng int >0, không trùng; `pointExpirationDays` int ≥1 (BR-253, BR-260) | `{ rules }` (cùng dạng GET). Trong 1 transaction: cập nhật bản active hoặc tạo mới (thresholds sort tăng dần) và ghi 1 dòng `gamification_point_rule_history` (`changedBy` = admin từ JWT, `before` = giá trị cũ hoặc null, `after`; cả hai có `pointExpirationDays`) (BR-258) | 400 `VALIDATION_ERROR` kèm `errors: [{field, message}]`; 401 thiếu user; 500 | `adminPatchPointRules()` → `gamification-config.service.ts > validatePointRulesInput(), upsertPointRules()` |
| GET | `/api/v1/admin/gamification/point-rules/history` | query `page` (mặc định 1), `limit` (mặc định 20, tối đa 100) | `{ items: [{ id, changedBy, changedByName, changedAt, before, after }], meta: { page, limit, total } }`, mới nhất trước; `changedByName` tra identity-service (`fetchUsersByIds()`), lỗi thì null | 500 | `adminGetPointRuleHistory()` → `listPointRuleHistory()` |

### 4.5 Nội bộ (3) — gateway không proxy

| Method | Path | Auth | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|
| GET | `/internal/v1/point-rules/campaign-completion` | header `x-internal-api-key` = `INTERNAL_REWARD_API_KEY` | — | `{ campaignCompletionPoints }` từ quy tắc đang hiệu lực | 401, 404 "No active point rules", 500 | `internal.routes.ts > GET /point-rules/campaign-completion` → `loadActivePointRules()` |
| GET | `/internal/v1/difficulties` (deprecated) | như trên | — | `{ difficulties: { difficulties: [...], total } }` (xem mục 9: lồng 2 cấp) | 401, 500 | `.../src/internal/internal.routes.ts > GET /difficulties` → `difficultyService.listActive()` (mặc định page 1, limit 20) |
| GET | `/internal/v1/difficulties/level/:level` (deprecated) | như trên | `level` int ≥1 | `{ difficulty }` | 400, 401, 404 "Difficulty not found", 500 | `internal.routes.ts > GET /difficulties/level/:level` → `findByLevel()` |

Người gọi: incident-service `ecolink-server/services/incident-service/src/modules/reward/reward-service.client.ts > getCampaignCompletionPoints()` (điền `greenPoints` của campaign). Hai route `/difficulties*` không còn người gọi. Comment trong code ghi rõ các endpoint HTTP enqueue green-point/facebook cũ đã bị gỡ.

### 4.6 Khác

| Method | Path | Handler |
|---|---|---|
| GET | `/health` | `.../src/index.ts` → `{ status: "ok", service: "reward-service" }` |
| — | Swagger/OpenAPI | `mountOpenApi()` trong `.../src/index.ts`; gateway phục vụ `/api-docs/specs/reward.json` |

## 5. Event / Job phát ra và lắng nghe

### 5.1 Hạ tầng queue

`.../src/queue/green-point-queue.bootstrap.ts` tạo 4 `SqsBackgroundJobQueue` (thư viện `ecolink-server/shared/da2-queue`) và một `BackgroundJobDispatcher`:

| Queue (env URL) | Store | Job type đăng ký trên dispatcher | Worker tiêu thụ |
|---|---|---|---|
| `SQS_GREEN_POINT_QUEUE_URL` | `RewardBackgroundJobStore` (bảng `reward_background_jobs`) | 3 job green point (bên dưới) | `GreenPointCreditWorker` |
| `SQS_FACEBOOK_RECOGNITION_QUEUE_URL` | `RewardBackgroundJobStore` | `CAMPAIGN_FACEBOOK_RECOGNITION` | `FacebookRecognitionWorker` |
| `SQS_REWARD_TRANSLATION_QUEUE_URL` | `RewardBackgroundJobStore` | `TRANSLATE_TEXT` | `TranslationWorker` |
| `SQS_REWARD_INTAKE_QUEUE_URL` | `NoopBackgroundJobStore` (không ghi DB) | (không đăng ký, chỉ tiêu thụ) | `RewardIntakeWorker` |

Envelope SQS: `{ jobId, version: 1, jobType, createdAt, payload }` (`ecolink-server/shared/da2-queue/src/infra/sqs-background-job-queue.ts > enqueue()`).

### 5.2 Sự kiện lắng nghe (consume)

| jobType | Nguồn | Payload | Xử lý | Bằng chứng |
|---|---|---|---|---|
| `REPORT_COMPLETION_GREEN_POINTS` | incident-service outbox → queue reward-intake | `{ reportId, userId }` (`points` của payload cũ bị bỏ qua) | `reportCompletionPoints` của quy tắc (0 = không cộng) → `REPORT_COMPLETION` (resourceType REPORT, khoá `report_completion:<reportId>`) → 1 dòng sổ + 1 lô; metadata `{ruleId, reportCompletionPoints}` | `.../green-point/strategies/report-completion-green-point.strategy.ts` |
| `CAMPAIGN_COMPLETION_GREEN_POINTS` | incident-service outbox | `{ campaignId, participants: [{userId, eligibleShifts (int ≥1), registeredShifts (int ≥0)}] }`; payload cũ `{ campaignId, credits: [{userId, points}] }` vẫn nhận trong một bản phát hành (dùng `points`, metadata `legacyPayload: true`) | Mỗi người `round(campaignCompletionPoints × min(1, eligibleShifts / max(1, registeredShifts)))`, 0 thì bỏ → `CAMPAIGN_COMPLETION` (resourceType CAMPAIGN, khoá `campaign_completion:<campaignId>`) → 1 dòng sổ + 1 lô; metadata `{ruleId, campaignCompletionPoints, eligibleShifts, registeredShifts}` | `.../strategies/campaign-completion-green-point.strategy.ts` |
| `REPORT_VOTE_MILESTONE_GREEN_POINTS` | incident-service outbox (vote.service) | `{ reportId, reportCreatorUserId, voteCount }` | Với mỗi threshold ≤ voteCount (sắp tăng dần) trong point rules active: points = `baseReportPoint × (index+1)`; resourceType `REPORT`, khoá `vote_milestone:<reportId>:<threshold>` → 1 dòng sổ + 1 lô; metadata `{ruleId, baseReportPoint, threshold, milestoneIndex, voteCountAtAward}` | `.../strategies/report-vote-milestone-green-point.strategy.ts > resolveReportVoteMilestoneCredits()` |
| `CAMPAIGN_FACEBOOK_RECOGNITION` | incident-service outbox | `{ campaignId, campaignTitle, recognizedUserIds[], completedAt, bannerUrl?, description?, recognizedVolunteers?[{name,email}] }` | Sinh caption (ai-service) → đăng Facebook Graph và/hoặc POST webhook | `.../facebook-recognition/facebook-recognition.service.ts > applyQueuedJob()` |
| `TRANSLATE_TEXT` | Chính reward-service (gift, difficulty) | `{ resourceType: GIFT\|DIFFICULTY, resourceId, translations[] }` | Gọi ai-service dịch, ghi đè cột vi/en | `.../queue/workers/translation.worker.ts` |

incident-service đẩy qua `SqsOutboxPublisher` (`ecolink-server/services/incident-service/src/outbox/outbox-publisher.ts`), `jobId` = id dòng outbox. `RewardIntakeWorker` định tuyến theo `jobType`: facebook → `facebookRecognitionService.applyQueuedJob()`, còn lại → `greenPointService.applyQueuedJob()` (`.../queue/workers/reward-intake.worker.ts`).

### 5.3 Job reward-service tự phát (produce)

| jobType | Ai enqueue | Khi nào |
|---|---|---|
| `TRANSLATE_TEXT` | `gift.service.ts > enqueueGiftTranslationJob()` (từ `create()`, `updateById()`), `difficulty.service.ts > enqueueDifficultyTranslationJob()` (từ `updateById()`) | Khi có text nguồn và thiếu ít nhất một ngôn ngữ |
| 3 job green point, `CAMPAIGN_FACEBOOK_RECOGNITION` | Có hàm `greenPointService.enqueue()`, `enqueueCampaignCompletionCredits()`, `facebookRecognitionService.enqueue()` nhưng **không có nơi nào gọi** trong reward-service | — |

### 5.4 HTTP gọi sang service khác

| Đích | Endpoint | Auth | Dùng ở |
|---|---|---|---|
| identity-service | `POST {IDENTITY_SERVICE_URL}/internal/v1/users/by-ids` body `{ ids }` (chia lô 100) | `x-internal-api-key: INTERNAL_IDENTITY_API_KEY` | `.../utils/identity-user.client.ts > fetchUsersByIds()`: admin redemptions (cả tên admin trong lịch sử đơn, spec 017), transactions (resource USER), lịch sử quy tắc điểm (`changedByName`) |
| notification-service | `POST {NOTIFICATION_SERVICE_URL}/api/v1/notifications/jobs` body `{ type: "website", kind: "GIFT_ORDER_SHIPPED", userId, payload }`, timeout 10 s, sau commit; lỗi chỉ log | `x-internal-api-key: INTERNAL_NOTIFICATION_API_KEY` | `.../utils/notification.client.ts > enqueueGiftOrderShippedNotification()` |
| incident-service | `GET {INCIDENT_SERVICE_URL}/api/v1/campaigns/by-ids?campaignIds=...`, `GET .../api/v1/reports/by-ids?reportIds=...` | Chuyển tiếp header `Authorization` của người gọi | `.../utils/incident-resource.client.ts` (trong `getTransactions()`) |
| ai-service | `POST {AI_SERVICE_URL}/internal/v1/translate` | `x-internal-api-key: INTERNAL_AI_API_KEY` | `.../modules/translation/translation.client.ts > translateText()` |
| ai-service | `POST {AI_SERVICE_URL}/api/v1/social/campaign-facebook-caption` (timeout 60 s) | Không gửi header auth | `facebook-recognition.service.ts > fetchCaptionFromAi()` |
| Facebook Graph | `POST https://graph.facebook.com/{version}/{objectId}/photos` (nếu `bannerUrl` là https) rồi fallback `/feed`; timeout 30 s; `appsecret_proof` HMAC-SHA256 nếu có app id + secret | access token | `postToFacebookGraph()` |
| Webhook tuỳ chọn | `POST FACEBOOK_RECOGNITION_WEBHOOK_URL` body `{ type, payload, generatedCaption? }` | — | `applyQueuedJob()` |

Mọi lỗi khi gọi identity/incident đều bị nuốt (log rồi trả Map rỗng) → dữ liệu `user`/`resource` = `null`, không làm fail request.

## 6. Job nền, cron, worker

Job hết hạn điểm (`.../modules/green-point/green-point-expiry.job.ts > startGreenPointExpiryJob()`, khởi động trong `src/worker.ts`, nên chạy cả trong API process): chạy ngay khi start rồi mỗi `GREEN_POINT_EXPIRY_INTERVAL_MS` (mặc định 1 giờ); `GREEN_POINT_EXPIRY_ENABLED=false` thì tắt. Mỗi lượt (`expireDueLots()`) duyệt theo lô 200 các lô `remaining > 0` và `expires_at <= now`; mỗi lô một transaction ngắn: ghi dòng `EXPIRE` −remaining (resourceType `GREEN_POINT_LOT`, khoá `expire:<lotId>`) rồi đặt `remaining = 0` có điều kiện (`remaining` chưa đổi). Lô lỗi được log và để lượt sau. Số dư không phụ thuộc job (lô quá hạn đã bị loại khi đọc), job chỉ giữ lịch sử đầy đủ (BR-261). Job hết hạn lô quà (`.../modules/gift/gift-stock-expiry.job.ts > startGiftStockExpiryJob()`, cũng khởi động trong `src/worker.ts`): chạy ngay khi start rồi mỗi `GIFT_STOCK_EXPIRY_INTERVAL_MS` (mặc định 24 giờ); `GIFT_STOCK_EXPIRY_ENABLED=false` thì tắt. Mỗi lượt: `expireDueBatches()` duyệt theo cursor các lô `remaining > 0` và `expires_at <= now`, mỗi lô một transaction (`remaining = 0` có điều kiện `remaining = R`, rồi movement `EXPIRED` −R); sau đó `reconcileStockCache()` đặt `gifts.stock_remaining` bằng tổng `remaining` lô còn hạn ở các quà đang lệch (BR-224, BR-235). Không có cron nào khác; bảng `season_schedule_rules` còn trong DB nhưng không còn code dùng.

Worker (`.../src/queue/register.ts > startAllQueues()`):

| Worker | Số instance (env, mặc định) |
|---|---|
| `GreenPointCreditWorker` | `GREEN_POINT_QUEUE_CONCURRENCY` (1) |
| `FacebookRecognitionWorker` | `FACEBOOK_QUEUE_CONCURRENCY` (1) |
| `TranslationWorker` | `TRANSLATION_QUEUE_CONCURRENCY` (1) |
| `RewardIntakeWorker` | `REWARD_INTAKE_QUEUE_CONCURRENCY` (2) |

Ngưỡng dùng chung: `WORKER_MAX_RECEIVE_COUNT` (5), `WORKER_RETRY_BASE_SECONDS` (30), `WORKER_MAX_RETRY_DELAY_SECONDS` (900), `WORKER_SQS_VISIBILITY_TIMEOUT_SECONDS` (120), `WORKER_BATCH_SIZE` (5), `WORKER_SQS_WAIT_TIME_SECONDS` (20).

Vòng xử lý (`ecolink-server/shared/da2-queue/src/core/queue-worker.ts > processMessage()`):
1. Parse envelope; thiếu `jobId`/`jobType` → coi là lỗi (đi nhánh retry).
2. `jobType` không khớp worker → `markFailed` + xoá message (không retry).
3. `markProcessing(jobId, receiveCount)`: store trả false (job đã COMPLETED/FAILED) → xoá message, bỏ qua.
4. `process()` thành công → xoá message → `markSucceeded`.
5. Lỗi: nếu `receiveCount >= maxRetries` → xoá message + `markFailed` (bỏ hẳn, không có DLQ trong code); ngược lại đặt visibility = `min(maxDelay, base × 2^(receiveCount−1))` giây và `markRetryScheduled`.
6. Lỗi poll → ngủ 2 s rồi poll lại.

Green point: `greenPointService.applyQueuedJob()` validate payload rồi chạy strategy trong một transaction `Serializable`; mỗi credit đi qua `green-point-ledger.util.ts > creditPoints()`: `insertLedgerOnce()` (`createMany skipDuplicates` = `INSERT … ON CONFLICT DO NOTHING` trên `(user_id, idempotency_key)`, không bắt P2002 nên transaction không bị abort) rồi tạo 1 lô cùng số điểm, `expires_at = now + pointExpirationDays` (BR-240, BR-260) (`.../modules/green-point/green-point.service.ts`). Strategy đọc quy tắc điểm trong chính transaction đó (`loadActivePointRules(tx)`); không có quy tắc nào → log cảnh báo, không cộng, job vẫn thành công (không retry) (`green-point-ledger.util.ts > warnNoPointRules()`).

Script chạy tay:
- `src/scripts/reconcile-green-points.ts` — `npm run reconcile:green-points` (bản build trong `dist`) hoặc `npm run dev:reconcile:green-points` (ts-node). Chạy `expireDueLots()` trước (còn lô lỗi thì dừng, exit 1), rồi với mỗi user có tổng sổ (dòng chưa xoá mềm) ≠ tổng `remaining` của lô, ghi một dòng `MIGRATION_ADJUST` = lô − sổ (resourceType USER, khoá `migration_adjust:<userId>`; **lô là chuẩn**) và ghi danh sách ra file CSV `reconcile-green-points-<thời điểm>.csv` ở thư mục chạy. Chạy lại an toàn (khoá idempotent). Thứ tự triển khai spec 014: backup DB → `prisma migrate deploy` (migration `20261022100000_green_point_lots`) → reconcile → bật code mới.

## 7. Phụ thuộc

| Phụ thuộc | Mục đích |
|---|---|
| PostgreSQL (`DATABASE_URL`, DB `rewarddb`) | Prisma |
| AWS SQS / LocalStack | 4 queue |
| identity-service | Hồ sơ user, `organizationIds` |
| incident-service | Chi tiết campaign/report cho lịch sử giao dịch; là producer của reward-intake; là consumer của `/internal/v1/point-rules/campaign-completion` |
| ai-service (Python) | Dịch vi/en; sinh caption Facebook |
| Facebook Graph API | Đăng bài vinh danh |
| Datadog (dd-trace) | APM (`.../src/tracer.ts`) |
| Thư viện nội bộ | `@da2/constants`, `@da2/queue`, `@da2/express-swagger` |

## 8. Biến môi trường

| Biến | Ý nghĩa | Nguồn |
|---|---|---|
| `PORT` | Cổng HTTP (mặc định 3002) | `src/index.ts` |
| `NODE_ENV` | Môi trường | `.env.example` |
| `DATABASE_URL` | Kết nối Postgres | `prisma/schema.prisma` |
| `JWT_SECRET` | Khoá verify JWT (fallback cứng nếu thiếu) | `utils/jwt.utils.ts` |
| `CORS_ORIGIN` | CORS origin (mặc định `*`, `credentials: true`) | `src/index.ts` |
| `SWAGGER_SERVER_URL` | URL server trong OpenAPI | `src/index.ts` |
| `INTERNAL_REWARD_API_KEY` | Khoá cho `/internal/v1` (phải khớp incident-service) | `middleware/internal-reward-auth.middleware.ts` |
| `IDENTITY_SERVICE_URL`, `INTERNAL_IDENTITY_API_KEY` | Gọi identity-service | `utils/identity-user.client.ts` |
| `NOTIFICATION_SERVICE_URL`, `INTERNAL_NOTIFICATION_API_KEY` | Thông báo `GIFT_ORDER_SHIPPED` khi đơn đổi quà được gửi (spec 017); thiếu thì bỏ qua và log cảnh báo ngoài production | `utils/notification.client.ts` |
| `INCIDENT_SERVICE_URL` | Gọi incident-service (không có trong `.env.example`; thiếu thì bỏ qua làm giàu resource) | `utils/incident-resource.client.ts` |
| `SQS_GREEN_POINT_QUEUE_URL`, `SQS_FACEBOOK_RECOGNITION_QUEUE_URL`, `SQS_REWARD_TRANSLATION_QUEUE_URL`, `SQS_REWARD_INTAKE_QUEUE_URL` | URL 4 queue; thiếu bất kỳ biến nào → throw khi import bootstrap (process không khởi động được) | `queue/reward-sqs-queue-factory.ts > createQueue()` |
| `AWS_GREENPOINT_ENDPOINT_URL` → `AWS_SQS_ENDPOINT` → `AWS_ENDPOINT_URL` | Endpoint SQS (ưu tiên theo thứ tự) | `queue/reward-sqs-queue-factory.ts > getSqsConfig()` |
| `AWS_REGION` | Region (mặc định us-east-1). `.env.example` lại khai báo `AWS_DEFAULT_REGION` | như trên |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Credential SQS (mặc định "test") | như trên |
| `WORKER_*` (6 biến), `*_QUEUE_CONCURRENCY` (4 biến) | Ngưỡng/đồng thời worker | `queue/register.ts` |
| `AI_SERVICE_URL`, `INTERNAL_AI_API_KEY` | ai-service (dịch, caption) | `translation.client.ts`, `facebook-recognition.service.ts` |
| `FACEBOOK_ACCESS_TOKEN`, `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET`, `FACEBOOK_FEED_OBJECT_ID` (mặc định `me`), `FACEBOOK_GRAPH_API_VERSION` (mặc định v21.0), `FACEBOOK_RECOGNITION_WEBHOOK_URL` | Đăng Facebook / webhook | `facebook-recognition.service.ts` |
| `DD_ENV`, `DD_VERSION` (+ `DD_AGENT_HOST` theo comment) | Datadog | `src/tracer.ts` |
| `GREEN_POINT_EXPIRY_INTERVAL_MS` (mặc định 3600000), `GREEN_POINT_EXPIRY_ENABLED` (`false` = tắt) | Job hết hạn điểm | `modules/green-point/green-point-expiry.job.ts` |
| `GIFT_STOCK_EXPIRY_INTERVAL_MS` (mặc định 86400000), `GIFT_STOCK_EXPIRY_ENABLED` (`false` = tắt) | Job hết hạn lô quà + đối soát cache tồn kho | `modules/gift/gift-stock-expiry.job.ts` |

## 9. Vấn đề cần xác nhận / [CHƯA HOÀN THIỆN]

**Bảo mật**
1. ~~`PATCH /api/v1/admin/gift-redemptions/:id/status` không có `requireAdmin`~~ — đã sửa 2026-10-21 (spec 015).
2. `JWT_SECRET` có fallback cứng `"fallback-secret-key"` — `utils/jwt.utils.ts`.
3. `.env.example` chứa giá trị trông như thật cho `FACEBOOK_APP_SECRET` và `FACEBOOK_ACCESS_TOKEN` (không chép lại ở đây) — cần thu hồi/thay placeholder.
4. `fetchCaptionFromAi()` gọi `/api/v1/social/campaign-facebook-caption` không gửi auth.
5. ~~`GET /metric-tables`, `/metric-columns` chỉ cần đăng nhập~~ — đã gỡ 2026-10-21 (spec 014) cùng module metrics.

**Bug / nghi bug**
6. `GET /internal/v1/difficulties` (deprecated, không còn người gọi) trả `data.difficulties = { difficulties, total }` (object) vì `listActive()` trả object; chỉ lấy 20 bản ghi đầu.
7. `difficulty.service.ts > updateById()`: khi body không có `name/nameVi/nameEn`, `nameVi`/`nameEn` = `""` (không phải `undefined`) nên luôn ghi đè thành chuỗi rỗng. Ví dụ PUT chỉ đổi `greenPoints` sẽ xoá tên vi/en.
8. ~~Huỷ đơn đổi quà không hoàn lại `stockRemaining`~~ — đã sửa 2026-10-21 (spec 015): huỷ đơn PROCESSING trả món về đúng lô (`returnToBatch()`, BR-234). ~~Huỷ đơn SHIPPED chưa hoàn kho~~ — đã sửa 2026-10-22 (spec 016): admin chọn `restock` khi huỷ đơn SHIPPED (BR-229).
9. ~~Đổi quà ghi `GIFT_REDEEM` âm nhưng không trừ `user_green_point_balances`; `GIFT_REDEEM_REFUND` bị tính vào `greenPointsEarnedTotal` và leaderboard legacy~~ — đã sửa 2026-10-21 (spec 014): số dư tính từ lô, `user_green_point_balances` không còn đọc/ghi; tổng đã kiếm chỉ cộng loại kiếm được (BR-266); leaderboard đã gỡ.
10. ~~`applyGreenPointLedgerCredit()` bắt P2002 rồi chạy tiếp trong transaction đã abort~~ — đã sửa 2026-10-21 (spec 014): ghi sổ qua `insertLedgerOnce()` (`ON CONFLICT DO NOTHING`), các util SP / RP đã xoá.
11. ~~Huỷ đơn tạo lô SP mới với hạn dùng mới~~ — đã sửa 2026-10-21 (spec 014): hoàn về đúng lô đã trừ, giữ hạn cũ (BR-263). Riêng đơn tạo trước khi có `green_point_lot_spends` vẫn nhận lô mới với hạn mới (BR-264).
12. `REPORT_VOTE_MILESTONE`: idempotency theo khoá `vote_milestone:<reportId>:<threshold>`; nếu admin đổi danh sách threshold, report cũ có thể được trả thêm cho threshold mới. Từ spec 014 dòng mốc vote có `resourceType = REPORT` (migration chuyển cả dòng cũ).
13. FacebookRecognition không idempotent: nếu đăng Facebook thành công nhưng webhook lỗi → job retry → đăng lại bài.
14. ~~`PATCH /admin/seasons/:id` đổi status không snapshot/payout~~ — đã gỡ 2026-10-21 (spec 014).
15. ~~`adminPatchPayoutTier()`, `adminPutMultiplier()` không validate~~ — đã gỡ 2026-10-21 (spec 014).
16. `PATCH point-rules` bắt buộc gửi đủ cả 5 field (ngữ nghĩa PUT). ~~Không chặn threshold âm/trùng; không xoá được `volunteerBonusCapByDifficulty`~~ — đã sửa 2026-10-21: mốc phải nguyên > 0, không trùng; cột bonus cap đã drop.
17. ~~`finalizeSeason()` không Serializable; tier ORG_AGGREGATE không bao giờ chi~~ — đã gỡ 2026-10-21 (spec 014).
18. ~~Rules AST của badge chỉ tính `user_point_transactions`~~ — đã gỡ 2026-10-21 (spec 014) cùng badge và metric metadata.
19. `GET /gifts` và `GET /difficulties` (deprecated) luôn trả `name: null` (và `description: null` cho gift), khác với field DTO.
20. API process (`index.ts` import `./worker`) cũng chạy worker; nếu chạy thêm `npm run worker` sẽ có 2 bộ consumer. Handler SIGTERM của worker gọi `process.exit(0)` cho cả API.
21. Seed `ecolink-server/scripts/seeds/reward-service-seed.sql` insert gift với `media_id` không tồn tại (không insert bảng `media`) → vi phạm FK `gifts_media_id_fkey`, cả transaction seed fail (đã chạy thử 2026-10-21, vẫn lỗi). ~~Số dư 150 không khớp giao dịch 100; đơn đổi quà không có dòng trừ điểm~~ — đã sửa 2026-10-21: seed không còn ghi `user_green_point_balances`, có dòng `GIFT_REDEEM` −100 nên sổ cộng về 0.
22. ~~Leaderboard theo `organizationId` tải toàn bộ bảng điểm~~ — đã gỡ 2026-10-21 (spec 014).

**[CHƯA HOÀN THIỆN]**
23. ~~Không có code cấp `UserBadgeGrant`; giảm giá theo badge luôn 0~~ — đã gỡ 2026-10-21 (spec 014): badge và giảm giá bị bỏ; đổi quà theo giá niêm yết (BR-226).
24. Các bảng ranking / season / badge (`seasons`, `user_point_transactions`, `user_season_rp_totals`, `leaderboard_snapshots`, `organization_season_scores`, `season_leaderboard_payout_tiers`, `volunteer_org_multiplier_rules`, `season_schedule_rules`, `spendable_point_rules`, `badge_definitions`, `user_badge_grants`, `report_milestone_awards`, `campaign_reward_awards`, `metric_tables`, `metric_columns`) cùng `user_green_point_balances` vẫn còn trong schema / DB nhưng không còn code đọc / ghi; drop ở đợt dọn dẹp.
25. Queue `green-point` và `facebook-recognition` có worker nhưng không có producer (HTTP enqueue cũ đã gỡ, xem comment trong `internal.routes.ts`). ~~Job `UPVOTE_ADDING_GREEN_POINTS`, `REFERRAL_ADDING_GREEN_POINTS` không có producer~~ — đã xoá hai strategy này 2026-10-21.
26. ~~`LeaderboardMetric` thừa giá trị; `badge-system-reference.md` lỗi thời~~ — đã gỡ 2026-10-21 (spec 014) (file đã xoá).
27. ~~Bảng `report_vote_green_point_rules` chỉ được đồng bộ để "ops/scripts"~~ — đã drop 2026-10-21 (migration `20261021100000_point_rules_earning`) cùng hàm đồng bộ.
28. Không có endpoint tạo/xoá Difficulty (deprecated), xoá Gift (chỉ `isActive`).
29. ~~`scripts/backfill-rp-totals.ts` import `rp-credit.util.ts` đã xoá~~ — đã xoá script cùng spec 014.
