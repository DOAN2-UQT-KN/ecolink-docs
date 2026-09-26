# 🏢 Flow tạo Organization (Tổ chức)

> Tài liệu mô tả **hiện trạng (as-is)** của luồng đăng ký tổ chức trên Ecolink, tính đến
> **2026-09-26** (Phase 1 của `ORG_OWNERSHIP_FLOW.md`). Thiết kế và lý do từng lựa chọn nằm ở
> `ORG_OWNERSHIP_FLOW.md`; phần "một tài khoản ORG duy nhất" của `REFACTOR_ORG_CREATION_FLOW.md`
> đã bị thay thế. Mọi đường dẫn file được ghi tương đối từ thư mục gốc `ecolink/`.

---

## 📌 Tóm tắt

Tổ chức **không được tạo trực tiếp**. Người nộp (không cần đăng nhập) lập một **hồ sơ đăng ký**
(`organization_applications`) kèm **danh sách owner**. Mỗi owner nhận một email riêng và phải tự
xác nhận; đủ xác nhận thì hồ sơ mới vào hàng chờ admin. Khi admin duyệt, hệ thống tạo bản ghi
`organizations` và **gắn vai cho từng owner** bằng `organization_members.role`.

- `User` luôn là **một con người**. Không còn tài khoản `accountType = ORG`.
- `Organization` **không đăng nhập**. Người thao tác cho tổ chức là các user có membership vai
  `LEGAL_REPRESENTATIVE` hoặc `OWNER`.
- Nguyên tắc: **không gán quyền cho một user khi chưa có bằng chứng người đó đồng ý** (bấm xác nhận
  từ hộp thư, có ghi IP và user agent).

### Năm trục trạng thái độc lập của tổ chức

| Trục | Trường | Giá trị | Ai điều khiển |
| --- | --- | --- | --- |
| Vòng đời tổ chức | `organizations.status` | `ACTIVE(1)` / `INACTIVE(2)` | Admin |
| Sở hữu hòm mail | `organizations.isEmailVerified` | bool | Kế thừa OTP **chỉ khi** email liên hệ = email người nộp |
| Loại chủ thể | `organizations.orgType` | `GOV`/`SCHOOL`/`CLUB`/`NGO`/`SOCIAL_ENTERPRISE` | User khai, admin xác nhận |
| Hồ sơ pháp lý | `organizations.kycStatus` | `NOT_SUBMITTED`/`APPROVED`/`EXPIRED`/`REVOKED` | Admin, qua hồ sơ |
| Mức tin cậy (Blue Tick) | `organizations.trustTier` | `NONE`/`BASIC`/`VERIFIED` | Admin |

---

## 🗺️ Sơ đồ luồng

```mermaid
sequenceDiagram
    participant U as Người nộp
    participant I as incident-service
    participant ID as identity-service
    participant N as notification-service
    participant O as Owner khác
    participant A as Admin

    Note over U,I: Bước 1 — OTP mở bản nháp
    U->>I: POST /organization-applications/email-otp { email }
    I--)N: ORG_APPLICATION_OTP (mã 6 số, 10 phút)
    U->>I: POST .../email-otp/verify { email, otp }
    I->>I: mở DRAFT (hoặc trả đơn đang mở) + owner = người nộp
    I-->>U: { application_id, tracking_token (180 ngày), resumed }

    Note over U,I: Soạn nháp (lưu nhiều lần)
    U->>I: POST /:id/documents/presign?token= → upload thẳng Cloudinary (private)
    U->>I: PUT /:id?token= { profile, channels, owners[], legal_representative, document_ids }

    Note over U,O: Nộp
    U->>I: POST /:id/submit?token=
    I->>ID: POST /internal/v1/users/lookup-by-emails (đình chỉ?)
    I->>I: trần 3 tổ chức, chống spam, email bị chặn
    I->>I: người nộp CONFIRMED; token 14 ngày cho owner khác
    I--)N: ORG_OWNER_CONFIRMATION_REQUEST (mỗi owner) + ORG_APPLICATION_RECEIVED

    Note over O,I: Owner xác nhận (không cần đăng nhập)
    O->>I: GET /owner-confirmations/:token
    alt Xác nhận
        O->>I: POST .../:token/confirm → lần cuối: PENDING_REVIEW
    else Tôi không liên quan
        O->>I: POST .../:token/decline → NEEDS_REVISION
        I--)N: ORG_OWNER_DECLINED → người nộp
    end

    Note over A,I: Thẩm định (chỉ thấy PENDING_REVIEW trở đi)
    A->>I: PUT /admin/.../:id/decision { APPROVE, lane, grant_blue_tick }
    I->>ID: POST /internal/v1/users/ensure (find-or-create)
    I->>I: 1 transaction: organizations + channels + memberships + outbox ORG_OWNER_ONBOARD
    I->>ID: (relay) POST /internal/v1/users/:id/activation-token
    I--)N: ACCOUNT_ACTIVATION (chưa có tài khoản) | ORG_OWNER_ATTACHED (đã có)
```

---

## 🧭 Bản đồ thành phần

| Lớp | Đường dẫn |
| --- | --- |
| Cổng OTP | `ecolink-client/app/(pages)/(main)/organizations/apply/page.tsx` |
| Trình soạn nháp | `.../organizations/apply/edit/page.tsx` (bước `StepOwners.tsx` mới) |
| Trang theo dõi | `.../organizations/apply/status/page.tsx` + `_components/OwnerConfirmations.tsx` |
| Trang owner xác nhận | `.../organizations/owner-confirm/page.tsx` |
| Kích hoạt tài khoản | `ecolink-client/app/(pages)/(auth)/activate-account/page.tsx` |
| Màn thẩm định | `ecolink-client/app/(pages)/(admin)/admin/organization-applications/` |
| Backend hồ sơ | `ecolink-server/services/incident-service/src/modules/organization_application/` |
| Membership | `.../modules/organization/organization-membership.service.ts`, `organization_member.repository.ts` |
| Onboarding sau duyệt | `.../organization_application/organization-owner-onboard.publisher.ts` |
| Sweeper hết hạn | `.../organization_application/owner-confirmation-expiry.job.ts` (chạy trong `src/worker.ts`) |
| Identity nội bộ | `ecolink-server/services/identity-service/src/internal/internal.routes.ts` |
| Email | `ecolink-server/services/notification-service/templates/notifications/{ORG_*,ACCOUNT_ACTIVATION}` |

---

# Phần 1 — Flow nghiệp vụ

### Bước 1 — Xác minh email và mở nháp

Vào `/organizations/apply`, nhập **email của chính mình** → nhận mã 6 số → nhập mã. Mã có hiệu
lực **10 phút**, sai tối đa **5 lần**, giới hạn **3 mã / email / giờ** và **10 mã / IP / giờ**.
Mã đúng mở một hồ sơ `DRAFT` (hoặc trả lại hồ sơ đang mở của email đó) và cấp **tracking token**
180 ngày. Trình duyệt chuyển sang `/organizations/apply/edit/:id?token=`.

### Bước 2 — Soạn nháp

Các bước: Hồ sơ → Liên hệ → **Owners** → Giấy tờ → Kiểm tra. Mỗi lần bấm "Tiếp tục" hoặc
"Lưu nháp", form được lưu (`PUT /:id`). Logo và ảnh bìa được upload Cloudinary (public preset)
**trước** khi lưu, nên nháp chỉ lưu URL.

- **Liên hệ:** email liên hệ công khai (mặc định là email người nộp) và ít nhất một kênh chính thức.
- **Owners:** 1–5 người (email, họ tên), đúng một **người đại diện pháp lý**. Người nộp luôn có
  tên và không gỡ được. KYC người đại diện (chức vụ, điện thoại, loại + số giấy tờ) chỉ dùng để
  thẩm định; số giấy tờ lưu **hash + 4 ký tự cuối**.
- **Giấy tờ:** tối đa **5 tệp**, mỗi tệp **≤ 10 MB**, `pdf | jpg | png`, lên kho riêng tư.

### Bước 3 — Nộp và chặn sớm

Trước khi gửi bất kỳ email nào, server kiểm: owner bị đình chỉ, owner đã làm owner 3 tổ chức,
email đang có tên ở ≥ 2 hồ sơ khác, email đã chặn lời mời, owner đã từ chối còn trong danh sách.
Người nộp tự được `CONFIRMED`. Còn owner chưa xác nhận → `AWAITING_OWNER_CONFIRMATION`; không còn
ai → thẳng `PENDING_REVIEW`.

### Bước 4 — Owner xác nhận

Mỗi owner nhận email tóm tắt hồ sơ (tổ chức, người nộp, các owner khác, vai của họ) và link tới
`/organizations/owner-confirm?token=`. Hạn **14 ngày**; người nộp gửi lại được **3 lần**, cách
nhau **≥ 1 giờ**.

- **Xác nhận:** ghi IP, user agent. Lần xác nhận cuối chuyển hồ sơ sang `PENDING_REVIEW` trong
  cùng transaction.
- **Tôi không liên quan:** hồ sơ về `NEEDS_REVISION`, người nộp nhận email; tuỳ chọn chặn email
  khỏi mọi lời mời sau này.
- **Hết hạn:** sweeper mỗi giờ đặt owner `EXPIRED`, hồ sơ về `NEEDS_REVISION`, người nộp nhận email.
- Đang đăng nhập bằng email khác email được mời → trang hiện cảnh báo.

### Bước 5 — Nộp lại

Sửa ở trình soạn nháp rồi nộp lại. Đổi **tên, loại tổ chức, người đại diện pháp lý hoặc danh sách
owner** → mọi xác nhận bị đặt lại (event `OWNER_CONFIRMATIONS_RESET`). Sửa mô tả, logo, giấy tờ,
kênh → giữ xác nhận. Owner bị gỡ chỉ bị đánh `removed_at`, không bị xoá.

### Bước 6 — Admin thẩm định

Tại `/admin/organization-applications` (chỉ thấy `PENDING_REVIEW`, `NEEDS_REVISION` và đã quyết định):

- **Claim** ghi reviewer, không đổi trạng thái.
- **Request more information** → `NEEDS_REVISION`.
- **Reject** → bắt buộc lý do.
- **Approve** → lane A / B, tuỳ chọn miễn giấy tờ (bắt buộc lý do) và cấp Blue Tick.
- Bảng owner cho từng người: giờ + IP xác nhận, tài khoản Ecolink (chưa có / hoạt động / đình chỉ /
  chưa kích hoạt), số tổ chức đang làm owner, cảnh báo **cùng IP trong 5 phút**.

### Bước 7 — Tổ chức ra đời và owner được gắn vai

Approve tạo tổ chức (`ACTIVE`, `kycStatus = APPROVED`) và membership `LEGAL_REPRESENTATIVE` /
`OWNER` cho từng owner. Owner chưa có tài khoản nhận email **kích hoạt** (72h, đặt mật khẩu ở
`/activate-account`); owner đã có tài khoản nhận email "đã được gắn làm owner". **Không gửi
mật khẩu tạm, không gửi link đặt lại mật khẩu.** Link kích hoạt hết hạn → tự gửi lại từ trang
đăng nhập.

### Vòng đời hồ sơ

```
DRAFT ──submit──> AWAITING_OWNER_CONFIRMATION ──đủ xác nhận──> PENDING_REVIEW ──decision──> APPROVED | REJECTED
  │                    │  owner từ chối / hết hạn                  │ request-info
  │                    └────────────> NEEDS_REVISION <──────────────┘
  │                                        └──submit──> AWAITING_OWNER_CONFIRMATION
WITHDRAWN ← người nộp tự rút, từ DRAFT / AWAITING / PENDING_REVIEW / NEEDS_REVISION
```

### ⚠️ Điểm dễ hiểu nhầm

1. **Mặc định mọi hồ sơ phải nộp giấy tờ.** Chỉ admin bấm miễn, bắt buộc lý do, ghi vào
   `organization_application_events`.
2. **Không auto-detect tên miền.** Lane do admin quyết.
3. **Owner đã có tài khoản Ecolink là hợp lệ.** Không còn edge case "email liên hệ đã có tài
   khoản": email liên hệ chỉ để liên hệ, không bao giờ thành tài khoản đăng nhập.
4. **"Xác nhận hết mới duyệt" nằm ở tầng trạng thái**: admin không nhìn thấy hồ sơ trước
   `PENDING_REVIEW`; lúc duyệt vẫn kiểm lại (phòng thủ theo chiều sâu).

---

# Phần 2 — Chi tiết kỹ thuật

## 2.1 API

| Method | Endpoint | Quyền |
| --- | --- | --- |
| POST | `/api/v1/organization-applications/email-otp` | public, rate-limited |
| GET | `/api/v1/organization-applications/email-otp/link?token=` | public |
| POST | `/api/v1/organization-applications/email-otp/verify` | public → mở DRAFT, trả tracking token |
| POST | `/api/v1/organization-applications/:id/documents/presign?token=` | tracking token, DRAFT / NEEDS_REVISION |
| GET | `/api/v1/organization-applications/:id?token=` | tracking token |
| PUT | `/api/v1/organization-applications/:id?token=` | lưu nháp, DRAFT / NEEDS_REVISION |
| POST | `/api/v1/organization-applications/:id/submit?token=` | nộp / nộp lại |
| POST | `/api/v1/organization-applications/:id/owners/:candidateId/resend?token=` | gửi lại lời mời |
| POST | `/api/v1/organization-applications/:id/withdraw?token=` | rút |
| GET | `/api/v1/organization-applications/:id/documents/:docId/file?token=` | tracking token, ghi audit |
| GET | `/api/v1/organization-applications/owner-confirmations/:token` | public (JWT tuỳ chọn để cảnh báo lệch email) |
| POST | `/api/v1/organization-applications/owner-confirmations/:token/confirm` | public |
| POST | `/api/v1/organization-applications/owner-confirmations/:token/decline` | public |
| GET | `/api/v1/admin/organization-applications` \| `/:id` | admin |
| GET | `.../:id/documents/:docId/file` | admin, **ghi audit mỗi lượt xem** |
| PUT | `.../:id/claim` \| `.../:id/request-info` \| `.../:id/decision` | admin |
| POST | `/internal/v1/users/lookup-by-emails` \| `/users/ensure` \| `/users/:id/activation-token` | internal (identity) |
| POST | `/api/v1/auth/activate-account` \| `/api/v1/auth/activation/resend` | public |
| POST | `/api/v1/organizations` | **internal-only**, tạo luôn membership OWNER |

## 2.2 Frontend

- `apply/_context/ApplicationContext.tsx` — hai chế độ: cổng OTP (`steps = ["email"]`) và trình
  soạn nháp (`profile, contact, owners, documents, review`). `saveDraft()` upload ảnh rồi `PUT`;
  `next()` validate bước hiện tại rồi lưu im lặng; `submit()` lưu rồi `POST /submit` và chuyển
  sang trang theo dõi.
- `apply/_services/application.service.ts` — `toSaveApplicationRequest()`, `validateOwnerList()`
  (cùng rule với server).
- `apply/_components/OwnerConfirmations.tsx` — thanh tiến trình `x/y đã xác nhận`, còn N ngày,
  nút "Gửi lại (còn n)", nút "Thay người" khi `NEEDS_REVISION`.
- `constants/apiErrorMessages.ts` — map mã lỗi (`OWNER_QUOTA_EXCEEDED`…) sang câu i18n, chèn email
  lấy từ message; dùng trong `hooks/reactQuery.ts > usePost`.
- `/organizations/apply/submitted` đã bị bỏ; nộp xong đi thẳng trang theo dõi.

## 2.3 Backend — module `organization_application`

| File | Vai trò |
| --- | --- |
| `organization-application-otp.service.ts` | OTP (lưu hash), tracking token |
| `organization-application.service.ts` | Mở nháp, lưu nháp (đồng bộ owner), nộp, gửi lại, rút, xem |
| `owner-candidates.ts` | `validateOwnerList`, snapshot + so sánh không phụ thuộc thứ tự key, sinh token |
| `owner-confirmation.service.ts` | Trang tóm tắt, xác nhận, từ chối, sweeper hết hạn |
| `organization-application-admin.service.ts` | Danh sách (ẩn DRAFT/AWAITING), claim, request-info, duyệt / từ chối |
| `organization-owner-onboard.publisher.ts` | Outbox `ORG_OWNER_ONBOARD`: email kích hoạt hoặc "đã gắn vai" |
| `identity-owner.client.ts` | HTTP client + circuit breaker sang identity |
| `storage/cloudinary-document-storage.ts` | Ký upload `type=authenticated`, stream file |

### Duyệt — vì sao chia hai nửa

User nằm ở DB identity, tổ chức và membership nằm ở DB incident:

1. **Trước transaction:** `ensure-users` tìm hoặc tạo user (status `PENDING_ACTIVATION`, không
   mật khẩu). Idempotent theo email; duyệt lỗi giữa chừng chỉ để lại user chưa kích hoạt, lần sau
   dùng lại.
2. **Một transaction ở incident:** khoá hồ sơ (`FOR UPDATE`), kiểm lại trạng thái và xác nhận,
   tạo `organizations` + channels, với từng owner lấy `pg_advisory_xact_lock` theo userId rồi đếm
   lại trần 3 tổ chức, tạo membership, ghi outbox `ORG_OWNER_ONBOARD` (dedupKey theo candidate).
3. **Relay:** publisher xin identity token kích hoạt (chỉ có khi user còn `PENDING_ACTIVATION`)
   rồi gửi email; lỗi thì relay retry với backoff.

### Bất biến ở tầng DB

Constraint trigger `DEFERRABLE INITIALLY DEFERRED` trên `organizations` (INSERT, UPDATE OF
`deleted_at`) và `organization_members` (UPDATE, DELETE): tổ chức chưa xoá phải còn ≥ 1 membership
vai owner, vỡ thì raise `ORG_MUST_HAVE_OWNER` lúc COMMIT.

## 2.4 Bảo vệ dữ liệu cá nhân

- Số CCCD/MSSV: lưu **sha256 + 4 ký tự cuối** (`legal_rep_id_hash`, `legal_rep_id_last4`).
- Giấy tờ: Cloudinary `type=authenticated`; admin xem qua proxy, mỗi lượt ghi `DOCUMENT_VIEWED`.
- Token xác nhận owner: chỉ lưu sha256; bản gốc chỉ nằm trong email.
- `confirm_ip`, `confirm_ua` lưu làm bằng chứng đồng thuận; chỉ admin thấy.
- Form có checkbox đồng ý, lưu `consented_at`.

> ⏳ **Chưa làm**: job purge giấy tờ + `legal_rep_*` sau 90 ngày (cột `purged_at` có sẵn).

## 2.5 Migration

```
incident-service:     20260926100000_org_multi_owner      ← ⚠️ DESTRUCTIVE (TRUNCATE ... CASCADE)
identity-service:     20260926100000_drop_org_accounts    ← ⚠️ xoá mọi user account_type = 'ORG'
notification-service: 20260926100000_notification_org_owner_kinds
```

Dữ liệu tổ chức / hồ sơ cũ là dữ liệu dev nên bị xoá, không chuyển đổi. Chạy xong phải
`npm run prisma:seed` lại.

## 2.6 Phạm vi chưa làm

- `ADD_OWNER` (cột `type` đã có), invitation nhẹ cho `ADMIN` / `CAMPAIGN_MANAGER` / `MEMBER`,
  bộ chọn ngữ cảnh tổ chức, `membershipVersion` — **[CHƯA HOÀN THIỆN]**.
- Luồng đi ra: thu hồi owner, owner tự rời, chuyển giao. Hiện owner không rời được; owner cuối cùng
  được DB bảo vệ.
- Blue Tick chưa mang đặc quyền nào (chưa nối campaign, chưa có sweep).

---

# Phần 3 — Kiểm thử

- Unit: `ecolink-server/services/incident-service/src/modules/organization_application/__tests__/`
  (`npx jest`).
- Integration: `src/__it__/org-ownership.it.test.ts` (Testcontainers, `npm run test:it`) — race
  trần 3 tổ chức, owner đã có tài khoản, trigger `ORG_MUST_HAVE_OWNER`. **Chưa chạy lần nào**
  (cần Docker).

Kịch bản thủ công (cần `npm run dev` + `npm run dev:worker` của incident và notification):

1. `/organizations/apply` → OTP → trình soạn nháp → thêm 2 owner → nộp.
2. Mở link trong email (hoặc bảng `notifications`, kind `ORG_OWNER_CONFIRMATION_REQUEST`): một
   người xác nhận, một người bấm "Tôi không liên quan" → hồ sơ về `NEEDS_REVISION`.
3. Thay người, nộp lại, đủ xác nhận → hồ sơ xuất hiện ở hàng chờ admin.
4. Duyệt → owner mới nhận `ACCOUNT_ACTIVATION`, owner cũ nhận `ORG_OWNER_ATTACHED`.
5. Đăng nhập bằng owner cũ, tạo chiến dịch cho tổ chức.
6. Trang đăng nhập với tài khoản chưa kích hoạt → "Gửi lại email kích hoạt".
