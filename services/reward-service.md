# reward-service

> Tài liệu viết hoàn toàn dựa trên code tại `ecolink-server/services/reward-service` (commit hiện tại). Đường dẫn bằng chứng tính từ `/Users/ngoc/ecolink`.
> Cơ chế dịch tự động (TRANSLATE_TEXT) được mô tả chi tiết riêng ở [translation-worker.md](./translation-worker.md).

## 1. Trách nhiệm của service

reward-service là nơi lưu trữ và tính toán mọi thứ liên quan đến điểm thưởng:

| Mảng | Nội dung | Bằng chứng |
|---|---|---|
| Difficulty | Bậc độ khó chiến dịch: giới hạn số tình nguyện viên (`maxVolunteers`) và điểm xanh thưởng (`greenPoints`). incident-service đọc qua API nội bộ | `ecolink-server/services/reward-service/prisma/schema.prisma > model Difficulty`, `.../src/internal/internal.routes.ts` |
| Green point (legacy) | Sổ cái `green_point_transactions` + số dư `user_green_point_balances`. Mỗi lần cộng điểm xanh sẽ đồng thời cộng SP và RP | `.../src/modules/green-point/green-point-ledger.util.ts > applyGreenPointLedgerCredit()` |
| Gamification v2 | Mùa giải (Season), điểm xếp hạng theo mùa CRP (công dân) / VRP (tình nguyện viên), ví điểm tiêu dùng SP chia theo lô FIFO có hạn dùng, bảng xếp hạng, snapshot cuối mùa, trả SP cho top theo payout tier, định nghĩa huy hiệu (badge) với rules AST | `.../src/modules/gamification/*` |
| Gift / đổi quà | Danh mục quà, đổi quà bằng SP (có giảm giá theo badge), đơn đổi quà với trạng thái vận chuyển, hoàn SP khi huỷ | `.../src/modules/gift/gift.service.ts` |
| Metric metadata | Danh mục bảng/cột metric cho UI dựng luật huy hiệu, dùng để validate `rulesConfig` | `.../src/modules/metrics/*` |
| Facebook recognition | Nhận sự kiện hoàn thành chiến dịch, sinh caption bằng ai-service, đăng lên Facebook Graph và/hoặc webhook | `.../src/modules/facebook-recognition/facebook-recognition.service.ts` |
| Dịch tự động | Dịch tên/mô tả Gift và tên Difficulty sang vi/en bất đồng bộ qua SQS | `.../src/queue/workers/translation.worker.ts` |

reward-service KHÔNG tự phát sự kiện sang service khác. Nó tiêu thụ hàng đợi `reward-intake` do incident-service đẩy vào (outbox) và gọi HTTP đồng bộ sang identity-service, incident-service, ai-service, Facebook Graph.

## 2. Cấu trúc thư mục / module chính

```
reward-service/
├── prisma/schema.prisma, migrations/, seed.ts, seeds/metric-metadata.seed.ts
├── scripts/backfill-rp-totals.ts            # backfill RP từ sổ green point (chạy tay)
└── src/
    ├── index.ts        # Express app, mount routes, import "./worker" => API process cũng chạy worker
    ├── worker.ts       # startAllQueues(), xử lý SIGINT/SIGTERM
    ├── tracer.ts       # dd-trace (Datadog)
    ├── config/prisma.client.ts
    ├── constants/http-status.ts             # re-export @da2/constants
    ├── middleware/     # auth, require-admin, internal-reward-auth, error
    ├── internal/internal.routes.ts          # /internal/v1/*
    ├── utils/          # identity-user.client.ts, incident-resource.client.ts, jwt.utils.ts
    ├── openapi/route-models.ts              # model cho swagger
    ├── queue/          # bootstrap (dispatcher + 4 SQS queue), register (khởi động worker), job stores, workers/
    └── modules/
        ├── difficulty/  gift/  user-points/  metrics/
        ├── green-point/ (service, factory, 5 strategy, ledger util, constants)
        ├── gamification/ (controller, routes, season, badge, badge-rule-evaluator, leaderboard, summary, config, campaign-reward-display, rp-credit, sp-credit, sp-wallet, point-source)
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

Tổng: **48 endpoint** = 45 endpoint dưới `/api/v1` (đều được api-gateway proxy) + 2 endpoint nội bộ `/internal/v1` (gateway KHÔNG proxy, chỉ incident-service gọi trực tiếp) + `GET /health` (không qua gateway).

Tất cả route `/api/v1/*` được mount trong `.../src/index.ts`. Path dưới đây là path đầy đủ qua gateway (gateway giữ nguyên path).

### 4.1 Difficulty (2)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/difficulties` | Không | Public | query `page` int ≥1 (mặc định 1), `limit` int 1..100 (mặc định 20) | `{ difficulties: DifficultyResponse[], page, limit, total, totalPages }`. Chỉ bản ghi `deletedAt = null`, sắp theo `level asc`. Trường `name` luôn `null`; `nameVi = nameVi ?? name` | 400 VALIDATION_ERROR, 500 | `.../modules/difficulty/difficulty.api.routes.ts > GET /difficulties` → `difficulty.service.ts > listActive()` |
| PUT | `/api/v1/difficulties/:id` | Bearer | admin | param `id` UUID; body tuỳ chọn: `name`, `nameVi`, `nameEn` (trim, 1..64), `maxVolunteers` (null hoặc int ≥1), `greenPoints` (int ≥0) | `{ difficulty }` | 400, 401, 403, 404 (không tìm thấy hoặc đã soft delete), 500 | `difficulty.api.routes.ts > PUT /difficulties/:id` → `difficulty.service.ts > updateById()` |

`updateById()` điền sẵn ngôn ngữ còn thiếu bằng text nguồn và enqueue job dịch (xem translation-worker.md). Không có endpoint tạo/xoá difficulty; dữ liệu đến từ seed.

### 4.2 Gift và đơn đổi quà (8)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/gifts` | Tuỳ chọn (`tryAuthenticate`) | Public; admin được lọc `isActive` | query: `page`≥1, `limit` 1..100, `search` (so khớp không phân biệt hoa thường trên `name`/`nameVi`/`nameEn`), `inStock` "true"/"false" (chỉ "true" có tác dụng: `stockRemaining` null hoặc >0), `isActive` "true"/"false" (chỉ admin), `greenPointsMin`/`greenPointsMax` int ≥0, `sortBy` ∈ createdAt/name/greenPoints (mặc định createdAt desc), `sortOrder` asc/desc | `{ gifts: GiftResponse[], page, limit, total, totalPages }`; `name` và `description` luôn `null`, kèm `media {id,url,type}` | 400 (validation; hoặc "greenPointsMin must be <= greenPointsMax"), 500 | `.../modules/gift/gift.api.routes.ts > GET /gifts` → `gift.service.ts > listGifts()` |
| GET | `/api/v1/gifts/:id` | Không | Public | `id` UUID | `{ gift }` (không lọc `isActive`, chỉ lọc `deletedAt null`) | 400, 404 "Gift not found", 500 | `gift.api.routes.ts > GET /gifts/:id` → `getGiftById()` |
| POST | `/api/v1/gifts` | Bearer | admin | body: `name` bắt buộc (1..255), `nameVi`/`nameEn` tuỳ chọn (1..255), `imageUrl` tuỳ chọn isURL, `description` bắt buộc string, `descriptionVi`/`descriptionEn` tuỳ chọn, `greenPoints` bắt buộc int ≥0, `stockRemaining` null hoặc int ≥0 (bỏ trống = không giới hạn), `isActive` boolean (mặc định true) | 201 `{ gift }` | 400, 401, 403, 500 | `POST /gifts` → `gift.service.ts > create()` |
| PUT | `/api/v1/gifts/:id` | Bearer | admin | như POST nhưng mọi field tuỳ chọn; `imageUrl` rỗng → gỡ media | `{ gift }` | 400, 401, 403, 404, 500 | `PUT /gifts/:id` → `updateById()` |
| POST | `/api/v1/gifts/:id/redeem` | Bearer | Mọi user | `id` UUID; body `phoneNumber` (trim, 7..32 ký tự, regex `^[0-9+\-()\s.]+$`), `pickupLocation` (trim, 1..1000) | `{ redemption: GiftRedemptionResponse }` (HTTP 200) | 400 (validation / "Missing id or user"), 401, 404 "Gift not found" (không tồn tại, đã xoá hoặc `isActive=false`), 422 "Gift is out of stock", 422 "Insufficient spendable points (SP)", 500 | `gift.api.routes.ts > handleGiftRedeemOrExchange()` → `gift.service.ts > redeem()` |
| POST | `/api/v1/gifts/:id/exchange` | Bearer | Mọi user | Giống hệt `/redeem` | Giống `/redeem` | Giống `/redeem` | `handleGiftRedeemOrExchange()` |
| GET | `/api/v1/admin/gift-redemptions` | Bearer | admin | query `page`, `limit` 1..100, `status` ∈ PROCESSING/SHIPPED/DELIVERED/CANCELLED, `sortBy` ∈ createdAt/greenPointsSpent/statusUpdatedAt (mặc định createdAt), `sortOrder` (mặc định desc) | `{ redemptions: AdminGiftRedemptionListItemResponse[] (kèm snapshot gift và user {id,name,avatar} lấy từ identity), page, limit, total, totalPages }` | 400, 401, 403, 500 | `GET /admin/gift-redemptions` → `listRedemptionsForAdmin()` |
| PATCH | `/api/v1/admin/gift-redemptions/:id/status` | Bearer | **Mọi user đã đăng nhập** (route KHÔNG có `requireAdmin`) | `id` UUID; body `status` ∈ 4 giá trị enum | `{ redemption }` | 400, 401, 404 "Redemption not found", 422 "Cannot change redemption status from X to Y", 500 | `PATCH /admin/gift-redemptions/:id/status` → `updateRedemptionStatus()` |

### 4.3 Điểm của tôi, leaderboard legacy, lịch sử đổi quà (5)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/me/points` | Bearer | Mọi user | — | `{ spendablePoints (SP còn hạn), greenPoints (số dư legacy), greenPointsEarnedTotal (tổng các dòng green point > 0), balance (= spendablePoints, deprecated) }` | 401, 500 | `.../modules/user-points/user-points.api.routes.ts > GET /me/points` → `user-points.service.ts > getPoints()` |
| GET | `/api/v1/me/points/transactions` | Bearer | Mọi user | query `page`, `limit` 1..100, `type` (string: "earned" → points>0, "spent" → points<0, giá trị khác → so khớp đúng `type`), `sortBy` ∈ createdAt/points/type, `sortOrder` | `{ transactions: (GreenPointTransaction + resource)[], page, limit, total, totalPages }`. `resource` được làm giàu: CAMPAIGN/REPORT gọi incident-service `/api/v1/campaigns/by-ids`, `/api/v1/reports/by-ids` (chuyển tiếp header Authorization), USER gọi identity-service, GIFT_REDEMPTION đọc DB cục bộ | 400, 401, 500 | `GET /me/points/transactions` → `getTransactions()` |
| GET | `/api/v1/leaderboard` | Không | Public | query `page`, `limit` 1..100 | `{ leaderboard: [{userId, greenPoints, user}], page, limit, total, totalPages }`. Xếp hạng theo SUM(points>0) trên `green_point_transactions` (raw SQL) | 400, 500 | `GET /leaderboard` → `getLeaderboard()` |
| GET | `/api/v1/leaderboard/me` | Bearer | Mọi user | — | `{ leaderboardMe: { rank (RANK() OVER), greenPoints } \| null }` | 401, 500 | `GET /leaderboard/me` → `getLeaderboardMe()` |
| GET | `/api/v1/me/redemptions` | Bearer | Mọi user | query `page`, `limit`, `sortBy` ∈ createdAt/greenPointsSpent/statusUpdatedAt, `sortOrder` | `{ redemptions: GiftRedemptionListItemResponse[], page, limit, total, totalPages }` | 400, 401, 500 | `GET /me/redemptions` → `gift.service.ts > listRedemptionsForUser()` |

### 4.4 Metric metadata (2)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/metric-tables` | Bearer | Mọi user | query `label` (contains, không phân biệt hoa thường) | `{ tables: MetricTableDto[] }` chỉ `isActive=true`, sắp theo `key` | 400, 401, 500 | `.../modules/metrics/metrics.api.routes.ts` → `metrics.controller.ts > getMetricTables()` |
| GET | `/api/v1/metric-columns` | Bearer | Mọi user | query `label`, `metricTableId` UUID | `{ columns: MetricColumnDto[] }` (kèm `metricTableKey`, `metricTableLabel`) | 400, 401, 500 | `metrics.controller.ts > getMetricColumns()` |

Không có endpoint ghi metric metadata; dữ liệu đến từ `prisma/seeds/metric-metadata.seed.ts`.

### 4.5 Gamification cho người dùng / public (9)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/seasons/current` | Không | Public | — | `{ season \| null }`. Ưu tiên season ACTIVE đang trong khung thời gian; nếu không có thì season ACTIVE mới nhất theo `startsAt`; nếu không có thì season mới nhất bất kỳ | 500 | `.../gamification/gamification.controller.ts > getSeasonCurrent()` → `season.service.ts > getCurrentSeason()` |
| GET | `/api/v1/seasons/:id` | Không | Public | `id` UUID | `{ season }` | 400, 404 "Season not found", 500 | `getSeasonById()` |
| GET | `/api/v1/me/gamification/summary` | Bearer | Mọi user | — | `{ season, rankingPoints {citizenRp, volunteerRp, totalRp}, spendablePoints {balance, nextExpiresAt}, legacyGreenPointsBalance }` | 401, 500 | `getGamificationSummary()` → `gamification-summary.service.ts > getSummaryForUser()` |
| GET | `/api/v1/me/gamification/point-transactions` | Bearer | Mọi user | query `page`, `limit` 1..100, `kind` ∈ CRP/VRP/SP | `{ transactions, total, page, limit, totalPages }` (sắp createdAt desc) | 400, 401, 500 | `getMyPointTransactions()` → `listPointTransactions()` |
| GET | `/api/v1/me/gamification/points-by-season` | Bearer | Mọi user | query `page`, `limit` | `{ seasons: [{seasonId,label,kind,status,startsAt,endsAt,crp,vrp,sp}], total, page, limit, totalPages }`; `sp` = tổng ròng các dòng SP có `createdAt` trong [startsAt, endsAt] | 400, 401, 500 | `getPointsBySeason()` → `gamification-summary.service.ts > getPointsBySeason()` |
| GET | `/api/v1/me/badges` | Bearer | Mọi user | query `seasonId` UUID (mặc định: season hiện tại) | `{ badges: [{id, grantedAt, metadata, season, badge{...}}] }` loại bỏ badge đã soft delete | 400, 401, 500 | `getMyBadges()` → `badge.service.ts > listMyBadges()` |
| GET | `/api/v1/gamification/campaign-reward-estimate` | Không | Public | query `difficultyLevel` int ≥1 (bắt buộc) | `{ difficultyLevel, basePoints (= difficulty.greenPoints), estimatedBonusMax (từ `volunteerBonusCapByDifficulty[level]`), estimatedRange{min,max}, difficultyName }` | 400, 404 "Difficulty not found", 500 | `getCampaignRewardEstimate()` → `campaign-reward-display.service.ts > getEstimateForDifficultyLevel()` |
| GET | `/api/v1/gamification/leaderboards/:metric` | Không | Public | `metric` ∈ crp/vrp/org_aggregate (không phân biệt hoa thường), query `page`, `limit` 1..100, `seasonId` UUID, `organizationId` UUID (chỉ lọc CRP/VRP theo thành viên tổ chức, dựa trên `organizationIds` do identity trả về) | `{ metric, seasonId, leaderboard, page, limit, total, totalPages }`. Season INACTIVE → đọc `leaderboard_snapshots`; season ACTIVE → đọc live `user_season_rp_totals` (chỉ điểm > 0) hoặc `organization_season_scores` | 400 ("metric must be crp, vrp, or org_aggregate"), 500 | `getGamificationLeaderboard()` → `gamification-leaderboard.service.ts > getLeaderboard()` |
| GET | `/api/v1/gamification/leaderboards/:metric/me` | Bearer | Mọi user | như trên (không có page/limit) | `{ leaderboardMe: {rank, score, seasonId} \| null }`; ORG_AGGREGATE luôn `null` | 400, 401, 500 | `getGamificationLeaderboardMe()` → `getLeaderboardMe()` |

### 4.6 Admin gamification config (15)

Mọi route: `authenticate` + `requireAdmin` (401/403). Body KHÔNG dùng express-validator; kiểm tra thủ công trong controller.

| Method | Path | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|
| GET | `/api/v1/admin/gamification/point-rules` | — | `{ rules \| null }` (bản ghi active mới nhất theo `effectiveFrom`) | 500 | `adminGetPointRules()` |
| PATCH | `/api/v1/admin/gamification/point-rules` | `baseReportPoint` int ≥0 (bắt buộc), `reportMilestoneThresholds` mảng int (bắt buộc), `volunteerBonusCapByDifficulty` JSON tuỳ chọn | `{ rules }`. Cập nhật bản active hoặc tạo mới; thresholds được sort tăng dần; đồng bộ sang bảng legacy `report_vote_green_point_rules` | 400 "Invalid baseReportPoint" / "reportMilestoneThresholds must be integer array", 500 | `adminPatchPointRules()` → `gamification-config.service.ts > upsertPointRules()`, `syncLegacyReportVoteGreenPointRules()` |
| GET | `/api/v1/admin/gamification/sp-rules` | — | `{ rules \| null }` | 500 | `adminGetSpRules()` |
| PATCH | `/api/v1/admin/gamification/sp-rules` | `expirationDays` int ≥1 | `{ rules }` | 400 "Invalid expirationDays", 500 | `adminPatchSpRules()` → `upsertSpRules()` |
| GET | `/api/v1/admin/gamification/multipliers` | — | `{ multipliers }` sắp isActive desc, priority desc | 500 | `adminListMultipliers()` |
| PUT | `/api/v1/admin/gamification/multipliers` | `code` (bắt buộc), `multiplier` number/string, `priority`, `isActive` | `{ multiplier }` upsert theo `code` | 400 "code required", 500 | `adminPutMultiplier()` → `upsertMultiplierRule()` |
| GET | `/api/v1/admin/gamification/season-schedules` | — | `{ schedules }` | 500 | `adminListSeasonSchedules()` |
| PUT | `/api/v1/admin/gamification/season-schedules` | `kind` MONTHLY/QUARTERLY (bắt buộc), `autoRotate`, `metadata` (null để xoá) | `{ schedule }` upsert theo `kind` | 400 "Invalid kind", 500 | `adminPutSeasonSchedule()` → `upsertSeasonScheduleRule()` |
| GET | `/api/v1/admin/gamification/payout-tiers` | query `seasonId` UUID (lọc `seasonId = x OR seasonId null`) | `{ tiers }` | 400, 500 | `adminListPayoutTiers()` |
| POST | `/api/v1/admin/gamification/payout-tiers` | `metric` ∈ CRP/VRP/ORG_AGGREGATE, `rankMin` int ≥1, `rankMax` int ≥ rankMin, `spAmount` int ≥0, `seasonId` string tuỳ chọn (null = tier mặc định) | 201 `{ tier }` | 400 "Invalid metric" / "Invalid tier bounds", 500 | `adminCreatePayoutTier()` |
| PATCH | `/api/v1/admin/gamification/payout-tiers/:id` | `id` UUID; `seasonId`, `metric`, `rankMin`, `rankMax`, `spAmount` (không validate) | `{ tier }` | 400 (id), 404, 500 | `adminPatchPayoutTier()` |
| DELETE | `/api/v1/admin/gamification/payout-tiers/:id` | `id` UUID | `{ deleted: true }` (xoá cứng) | 400, 404, 500 | `adminDeletePayoutTier()` |
| GET | `/api/v1/admin/gamification/badges` | query `includeInactive` "true"/"false". "true" → trả cả bản ghi đã soft delete; mặc định chỉ `deletedAt null` | `{ badges }` | 400, 500 | `adminListBadges()` → `listDefinitionsAdmin()` |
| POST | `/api/v1/admin/gamification/badges` | `name` (bắt buộc), `category` ∈ REPORT/CAMPAIGN/CONTRIBUTION/RANK (bắt buộc), `slug` (tuỳ chọn, tự sinh snake_case từ name, tự thêm hậu tố `_2`…`_64` nếu trùng), `symbol`, `scope` LIFETIME/SEASON, `rulesConfig` object (hoặc legacy `ruleType` THRESHOLD/RANK + `metric` CRP/VRP/ORG_AGGREGATE + `threshold` ≥0 / `rankTopN` ≥1), `isRepeatable`, `cooldownSeconds` int ≥0, `maxGrantsPerUser` int >0 hoặc null, `reward` object (key cho phép: discountBps/discount_bps 0..10000, bonus_sp ≥0, partner_tier_codes string[], perks object), `isActive`, `publishedAt` | 201 `{ badge }` | 400 VALIDATION_ERROR (không kèm chi tiết), 500 | `adminCreateBadge()` → `badge.service.ts > createDefinition()` |
| PATCH | `/api/v1/admin/gamification/badges/:id` | `id` UUID; các field như POST (trừ slug), thêm `deletedAt` (soft delete/khôi phục); `rulesConfig: null` xoá luật; `publishedAt` chỉ set được, không gỡ được | `{ badge }` | 400 (kể cả khi đổi `category`/`scope` sau khi đã có grant), 404, 500 | `adminPatchBadge()` → `patchDefinition()` |

### 4.7 Admin seasons (4)

| Method | Path | Auth | Role | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|---|
| GET | `/api/v1/admin/seasons` | Bearer | admin | query `page`, `limit` 1..100, `kind` MONTHLY/QUARTERLY, `search` (contains trên `label`) | `{ seasons, total, page, limit, totalPages }` | 400, 401, 403, 500 | `adminListSeasons()` → `season.service.ts > listAdmin()` |
| POST | `/api/v1/admin/seasons` | Bearer | admin | `kind` MONTHLY/QUARTERLY, `startsAt`, `endsAt` (parse được Date), `label`, `status` (1/2 hoặc "ACTIVE"/"INACTIVE"; mặc định ACTIVE) | 201 `{ season }` | 400 "Invalid kind" / "Invalid dates" / "Invalid status", 500 | `adminCreateSeason()` → `createSeason()` |
| PATCH | `/api/v1/admin/seasons/:id` | Bearer | admin | `id` UUID; `label`, `startsAt`, `endsAt`, `status`, `kind` | `{ season }` | 400 "Invalid status", 404, 500 | `adminPatchSeason()` → `patchSeason()` |
| POST | `/api/v1/admin/seasons/:id/finalize` | Bearer | admin | `id` UUID; query `openNext` boolean; body `startsAt`, `endsAt` (bắt buộc và startsAt < endsAt khi openNext=true), `nextLabel` | `{ snapshotsWritten, closed?, next? }` | 400 ("startsAt is required when openNext=true", "endsAt is required…", "startsAt must be earlier than endsAt"), 404 "Season not found", 422 "Season must be 1 (ACTIVE) or 2 (INACTIVE)", 422 "Another ACTIVE season already exists", 500 | `adminFinalizeSeason()` → `season.service.ts > finalizeSeason()` |

### 4.8 Nội bộ (2) — gateway không proxy

| Method | Path | Auth | Request | Response | Mã lỗi | Handler |
|---|---|---|---|---|---|---|
| GET | `/internal/v1/difficulties` | header `x-internal-api-key` = `INTERNAL_REWARD_API_KEY` | — | `{ difficulties: { difficulties: [...], total } }` (xem mục 9: lồng 2 cấp) | 401, 500 | `.../src/internal/internal.routes.ts > GET /difficulties` → `difficultyService.listActive()` (mặc định page 1, limit 20) |
| GET | `/internal/v1/difficulties/level/:level` | như trên | `level` int ≥1 | `{ difficulty }` | 400, 401, 404 "Difficulty not found", 500 | `internal.routes.ts > GET /difficulties/level/:level` → `findByLevel()` |

Người gọi: incident-service `ecolink-server/services/incident-service/src/modules/reward/reward-service.client.ts > getDifficulties(), getDifficultyByLevel()` (dùng khi kiểm tra sức chứa chiến dịch). Comment trong code ghi rõ các endpoint HTTP enqueue green-point/facebook cũ đã bị gỡ.

### 4.9 Khác

| Method | Path | Handler |
|---|---|---|
| GET | `/health` | `.../src/index.ts` → `{ status: "ok", service: "reward-service" }` |
| — | Swagger/OpenAPI | `mountOpenApi()` trong `.../src/index.ts`; gateway phục vụ `/api-docs/specs/reward.json` |

## 5. Event / Job phát ra và lắng nghe

### 5.1 Hạ tầng queue

`.../src/queue/green-point-queue.bootstrap.ts` tạo 4 `SqsBackgroundJobQueue` (thư viện `ecolink-server/shared/da2-queue`) và một `BackgroundJobDispatcher`:

| Queue (env URL) | Store | Job type đăng ký trên dispatcher | Worker tiêu thụ |
|---|---|---|---|
| `SQS_GREEN_POINT_QUEUE_URL` | `RewardBackgroundJobStore` (bảng `reward_background_jobs`) | 5 job green point (bên dưới) | `GreenPointCreditWorker` |
| `SQS_FACEBOOK_RECOGNITION_QUEUE_URL` | `RewardBackgroundJobStore` | `CAMPAIGN_FACEBOOK_RECOGNITION` | `FacebookRecognitionWorker` |
| `SQS_REWARD_TRANSLATION_QUEUE_URL` | `RewardBackgroundJobStore` | `TRANSLATE_TEXT` | `TranslationWorker` |
| `SQS_REWARD_INTAKE_QUEUE_URL` | `NoopBackgroundJobStore` (không ghi DB) | (không đăng ký, chỉ tiêu thụ) | `RewardIntakeWorker` |

Envelope SQS: `{ jobId, version: 1, jobType, createdAt, payload }` (`ecolink-server/shared/da2-queue/src/infra/sqs-background-job-queue.ts > enqueue()`).

### 5.2 Sự kiện lắng nghe (consume)

| jobType | Nguồn | Payload | Xử lý | Bằng chứng |
|---|---|---|---|---|
| `REPORT_COMPLETION_GREEN_POINTS` | incident-service outbox → queue reward-intake | `{ reportId, userId, points }` | Cộng điểm xanh `REPORT_COMPLETION` (resourceType REPORT) → SP + CRP | `.../green-point/strategies/report-completion-green-point.strategy.ts` |
| `CAMPAIGN_COMPLETION_GREEN_POINTS` | incident-service outbox | `{ campaignId, credits: [{userId, points}] }` | Mỗi credit → `CAMPAIGN_COMPLETION` (resourceType CAMPAIGN) → SP + VRP | `.../strategies/campaign-completion-green-point.strategy.ts` |
| `REPORT_VOTE_MILESTONE_GREEN_POINTS` | incident-service outbox (vote.service) | `{ reportId, reportCreatorUserId, voteCount }` | Với mỗi threshold ≤ voteCount trong point rules active: points = `baseReportPoint × (index+1)`; resourceType `REPORT_VOTE_MILESTONE_<threshold>` → SP + CRP | `.../strategies/report-vote-milestone-green-point.strategy.ts`, `gamification-config.service.ts > resolveReportVoteMilestoneCredits()` |
| `UPVOTE_ADDING_GREEN_POINTS` | Không tìm thấy producer trong incident/identity/notification | `{ userId, points, resourceId, resourceType }` | `UPVOTE` → SP + CRP | `.../strategies/upvote-adding-green-point.strategy.ts` |
| `REFERRAL_ADDING_GREEN_POINTS` | Không tìm thấy producer | `{ userId, points, resourceId }` | `REFERRAL` (resourceType USER) → SP + CRP | `.../strategies/referral-adding-green-point-service.strategy.ts` |
| `CAMPAIGN_FACEBOOK_RECOGNITION` | incident-service outbox | `{ campaignId, campaignTitle, recognizedUserIds[], completedAt, bannerUrl?, description?, recognizedVolunteers?[{name,email}] }` | Sinh caption (ai-service) → đăng Facebook Graph và/hoặc POST webhook | `.../facebook-recognition/facebook-recognition.service.ts > applyQueuedJob()` |
| `TRANSLATE_TEXT` | Chính reward-service (gift, difficulty) | `{ resourceType: GIFT\|DIFFICULTY, resourceId, translations[] }` | Gọi ai-service dịch, ghi đè cột vi/en | `.../queue/workers/translation.worker.ts` |

incident-service đẩy qua `SqsOutboxPublisher` (`ecolink-server/services/incident-service/src/outbox/outbox-publisher.ts`), `jobId` = id dòng outbox. `RewardIntakeWorker` định tuyến theo `jobType`: facebook → `facebookRecognitionService.applyQueuedJob()`, còn lại → `greenPointService.applyQueuedJob()` (`.../queue/workers/reward-intake.worker.ts`).

### 5.3 Job reward-service tự phát (produce)

| jobType | Ai enqueue | Khi nào |
|---|---|---|
| `TRANSLATE_TEXT` | `gift.service.ts > enqueueGiftTranslationJob()` (từ `create()`, `updateById()`), `difficulty.service.ts > enqueueDifficultyTranslationJob()` (từ `updateById()`) | Khi có text nguồn và thiếu ít nhất một ngôn ngữ |
| 5 job green point, `CAMPAIGN_FACEBOOK_RECOGNITION` | Có hàm `greenPointService.enqueue()`, `enqueueCampaignCompletionCredits()`, `facebookRecognitionService.enqueue()` nhưng **không có nơi nào gọi** trong reward-service | — |

### 5.4 HTTP gọi sang service khác

| Đích | Endpoint | Auth | Dùng ở |
|---|---|---|---|
| identity-service | `POST {IDENTITY_SERVICE_URL}/internal/v1/users/by-ids` body `{ ids }` (chia lô 100) | `x-internal-api-key: INTERNAL_IDENTITY_API_KEY` | `.../utils/identity-user.client.ts > fetchUsersByIds()`: leaderboard, admin redemptions, transactions (resource USER), lọc theo `organizationId` |
| incident-service | `GET {INCIDENT_SERVICE_URL}/api/v1/campaigns/by-ids?campaignIds=...`, `GET .../api/v1/reports/by-ids?reportIds=...` | Chuyển tiếp header `Authorization` của người gọi | `.../utils/incident-resource.client.ts` (trong `getTransactions()`) |
| ai-service | `POST {AI_SERVICE_URL}/internal/v1/translate` | `x-internal-api-key: INTERNAL_AI_API_KEY` | `.../modules/translation/translation.client.ts > translateText()` |
| ai-service | `POST {AI_SERVICE_URL}/api/v1/social/campaign-facebook-caption` (timeout 60 s) | Không gửi header auth | `facebook-recognition.service.ts > fetchCaptionFromAi()` |
| Facebook Graph | `POST https://graph.facebook.com/{version}/{objectId}/photos` (nếu `bannerUrl` là https) rồi fallback `/feed`; timeout 30 s; `appsecret_proof` HMAC-SHA256 nếu có app id + secret | access token | `postToFacebookGraph()` |
| Webhook tuỳ chọn | `POST FACEBOOK_RECOGNITION_WEBHOOK_URL` body `{ type, payload, generatedCaption? }` | — | `applyQueuedJob()` |

Mọi lỗi khi gọi identity/incident đều bị nuốt (log rồi trả Map rỗng) → dữ liệu `user`/`resource` = `null`, không làm fail request.

## 6. Job nền, cron, worker

Không có cron/scheduler nào trong code (bảng `season_schedule_rules` chỉ lưu cấu hình, không có code dùng `autoRotate`).

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

Green point: `greenPointService.applyQueuedJob()` validate payload rồi chạy strategy trong một transaction `Serializable` (`.../modules/green-point/green-point.service.ts`).

Script chạy tay: `scripts/backfill-rp-totals.ts` → `rp-credit.util.ts > backfillRankingPointsFromGreenLedger()` (tạo RP cho mọi dòng green point dương, idempotent theo `idempotencyKey`).

## 7. Phụ thuộc

| Phụ thuộc | Mục đích |
|---|---|
| PostgreSQL (`DATABASE_URL`, DB `rewarddb`) | Prisma |
| AWS SQS / LocalStack | 4 queue |
| identity-service | Hồ sơ user, `organizationIds` |
| incident-service | Chi tiết campaign/report cho lịch sử giao dịch; là producer của reward-intake; là consumer của `/internal/v1/difficulties` |
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
| `INCIDENT_SERVICE_URL` | Gọi incident-service (không có trong `.env.example`; thiếu thì bỏ qua làm giàu resource) | `utils/incident-resource.client.ts` |
| `SQS_GREEN_POINT_QUEUE_URL`, `SQS_FACEBOOK_RECOGNITION_QUEUE_URL`, `SQS_REWARD_TRANSLATION_QUEUE_URL`, `SQS_REWARD_INTAKE_QUEUE_URL` | URL 4 queue; thiếu bất kỳ biến nào → throw khi import bootstrap (process không khởi động được) | `queue/reward-sqs-queue-factory.ts > createQueue()` |
| `AWS_GREENPOINT_ENDPOINT_URL` → `AWS_SQS_ENDPOINT` → `AWS_ENDPOINT_URL` | Endpoint SQS (ưu tiên theo thứ tự) | `queue/reward-sqs-queue-factory.ts > getSqsConfig()` |
| `AWS_REGION` | Region (mặc định us-east-1). `.env.example` lại khai báo `AWS_DEFAULT_REGION` | như trên |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Credential SQS (mặc định "test") | như trên |
| `WORKER_*` (6 biến), `*_QUEUE_CONCURRENCY` (4 biến) | Ngưỡng/đồng thời worker | `queue/register.ts` |
| `AI_SERVICE_URL`, `INTERNAL_AI_API_KEY` | ai-service (dịch, caption) | `translation.client.ts`, `facebook-recognition.service.ts` |
| `FACEBOOK_ACCESS_TOKEN`, `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET`, `FACEBOOK_FEED_OBJECT_ID` (mặc định `me`), `FACEBOOK_GRAPH_API_VERSION` (mặc định v21.0), `FACEBOOK_RECOGNITION_WEBHOOK_URL` | Đăng Facebook / webhook | `facebook-recognition.service.ts` |
| `DD_ENV`, `DD_VERSION` (+ `DD_AGENT_HOST` theo comment) | Datadog | `src/tracer.ts` |

## 9. Vấn đề cần xác nhận / [CHƯA HOÀN THIỆN]

**Bảo mật**
1. `PATCH /api/v1/admin/gift-redemptions/:id/status` không có `requireAdmin` (comment ghi "@access Private") → mọi user đã đăng nhập có thể đổi trạng thái đơn của bất kỳ ai (SHIPPED/DELIVERED/CANCELLED, kích hoạt hoàn SP) — `gift.api.routes.ts`.
2. `JWT_SECRET` có fallback cứng `"fallback-secret-key"` — `utils/jwt.utils.ts`.
3. `.env.example` chứa giá trị trông như thật cho `FACEBOOK_APP_SECRET` và `FACEBOOK_ACCESS_TOKEN` (không chép lại ở đây) — cần thu hồi/thay placeholder.
4. `fetchCaptionFromAi()` gọi `/api/v1/social/campaign-facebook-caption` không gửi auth.
5. `GET /metric-tables`, `/metric-columns` chỉ cần đăng nhập (dù phục vụ UI admin).

**Bug / nghi bug**
6. `GET /internal/v1/difficulties` trả `data.difficulties = { difficulties, total }` (object) vì `listActive()` trả object; incident-service `readDifficultiesFromResponse()` chờ mảng → luôn nhận `[]`. Ngoài ra chỉ lấy 20 bản ghi đầu.
7. `difficulty.service.ts > updateById()`: khi body không có `name/nameVi/nameEn`, `nameVi`/`nameEn` = `""` (không phải `undefined`) nên luôn ghi đè thành chuỗi rỗng. Ví dụ PUT chỉ đổi `greenPoints` sẽ xoá tên vi/en.
8. Huỷ đơn đổi quà không hoàn lại `stockRemaining` — `updateRedemptionStatus()`.
9. Đổi quà ghi dòng `GIFT_REDEEM` âm vào `green_point_transactions` nhưng không trừ `user_green_point_balances`; hoàn tiền ghi dòng dương `GIFT_REDEEM_REFUND` → dòng hoàn tiền bị tính vào `greenPointsEarnedTotal` và leaderboard legacy (`SUM(points>0)`).
10. `applyGreenPointLedgerCredit()` bắt lỗi P2002 rồi tiếp tục trong cùng transaction Postgres. Sau lỗi unique, Postgres đánh dấu transaction aborted, các câu lệnh tiếp theo (ví dụ credit tiếp theo trong batch campaign) có thể fail → cả job retry và bị bỏ sau 5 lần. Tương tự trong `applySpendablePointCredit()`/`applyRankingPointCredit()`. Cần xác nhận hành vi thực tế với Prisma.
11. Huỷ đơn tạo lô SP mới với hạn dùng mới (không khôi phục lô cũ).
12. `REPORT_VOTE_MILESTONE`: idempotency theo `REPORT_VOTE_MILESTONE_<threshold>`; nếu admin đổi danh sách threshold, report cũ có thể được trả thêm cho threshold mới. `mapResourceTypeToSourceType()` map resourceType này thành `SYSTEM` thay vì `REPORT`.
13. FacebookRecognition không idempotent: nếu đăng Facebook thành công nhưng webhook lỗi → job retry → đăng lại bài.
14. `PATCH /admin/seasons/:id` cho đổi `status` ACTIVE→INACTIVE mà không freeze snapshot/payout; cũng cho INACTIVE→ACTIVE. Không validate `startsAt < endsAt`, không kiểm tra trùng season ACTIVE ở create/patch (chỉ finalize kiểm tra). Date/kind không hợp lệ → lỗi Prisma → 500.
15. `adminPatchPayoutTier()`, `adminPutMultiplier()` không validate giá trị (NaN, metric sai) → 500.
16. `PATCH point-rules` bắt buộc gửi đủ cả 2 field (ngữ nghĩa PUT); không thể xoá `volunteerBonusCapByDifficulty` (`null ?? undefined`); không chặn threshold âm/trùng.
17. `seasonService.finalizeSeason()` dùng transaction mặc định (không Serializable), payout tier chỉ áp cho CRP/VRP; tier ORG_AGGREGATE tạo được nhưng không bao giờ chi.
18. Rules AST: evaluator chỉ tính target `user_point_transactions` (cộng mọi kind CRP+VRP+SP, bỏ qua `metric`/`ruleType` legacy); `orders/reviews/reports/votes` luôn false; target khác trong metric seed (`campaigns`, `green_point_transactions`…) qua được validate nhưng evaluator coi giá trị = 0. SUM trên field `id` (có trong seed) sẽ lỗi ở Prisma.
19. `GET /gifts` và `GET /difficulties` luôn trả `name: null` (và `description: null` cho gift), khác với field DTO.
20. API process (`index.ts` import `./worker`) cũng chạy worker; nếu chạy thêm `npm run worker` sẽ có 2 bộ consumer. Handler SIGTERM của worker gọi `process.exit(0)` cho cả API.
21. Seed `ecolink-server/scripts/seeds/reward-service-seed.sql` insert gift với `media_id` không tồn tại (không insert bảng `media`) → vi phạm FK `gifts_media_id_fkey`, cả transaction seed có thể fail. Seed cũng tạo số dư 150 không khớp giao dịch 100, và đơn đổi quà không có dòng trừ điểm.
22. Leaderboard có `organizationId` tải toàn bộ bảng điểm của season rồi gọi identity cho tất cả user (hiệu năng).

**[CHƯA HOÀN THIỆN]**
23. Không có code nào tạo `UserBadgeGrant`; `badgeService.evaluateBadge()` không được gọi ở đâu → huy hiệu không bao giờ được cấp; `getBestStoreDiscountBps()` thực tế luôn 0; `slugLockedAt` chỉ được set khi publish.
24. Không có code ghi `organization_season_scores`, `report_milestone_awards`, `campaign_reward_awards`; `volunteer_org_multiplier_rules` và `season_schedule_rules` chỉ là CRUD cấu hình, không được worker đọc. `reward.bonus_sp`, `partner_tier_codes`, `perks` chỉ được validate, không dùng.
25. Queue `green-point` và `facebook-recognition` có worker nhưng không có producer (HTTP enqueue cũ đã gỡ, xem comment trong `internal.routes.ts`). Job `UPVOTE_ADDING_GREEN_POINTS`, `REFERRAL_ADDING_GREEN_POINTS` không có producer trong incident/identity/notification.
26. `LeaderboardMetric` có `REPORT_UPVOTES`, `REPORT_COUNT`, `CAMPAIGN_COMPLETED` nhưng API chỉ chấp nhận crp/vrp/org_aggregate. `src/modules/gamification/badge-system-reference.md` mô tả mô hình badge cũ (ruleType/metric/threshold, upsert theo (userId,badgeId,seasonId)) không còn khớp schema.
27. Bảng `report_vote_green_point_rules` chỉ được đồng bộ để "ops/scripts"; worker đọc `gamification_point_rules`.
28. Không có endpoint tạo/xoá Difficulty, xoá Gift (chỉ `isActive`), ghi metric metadata.
