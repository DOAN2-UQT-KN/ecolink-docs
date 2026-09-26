# 05 — Xác thực và phân quyền

> Đường dẫn viết tắt giống [02-business-flows.md](02-business-flows.md).

## 1. Cơ chế xác thực

### 1.1 JWT (người dùng)

| Mục | Hành vi trong code | Bằng chứng |
|---|---|---|
| Phát hành | identity ký **access token** và **refresh token** bằng cùng `JWT_SECRET`, cùng payload `{userId, email, role}`, không có claim phân biệt loại token. TTL: `JWT_EXPIRES_IN` (code mặc định 30m) và `JWT_REFRESH_EXPIRES_IN` (30d) | `ID/utils/jwt.utils.ts`, `ID/modules/auth/auth.service.ts > login()` |
| Claim `role` | **Tên role** (`ADMIN`, `USER`) cả khi login lẫn khi refresh (bug refresh ghi roleId đã sửa 2026-09-26) | `auth.service.ts > login(), refreshAccessToken()` |
| Lưu phía server | Chỉ lưu `sha256(refresh token)` ở `auth_tokens` (type REFRESH). Access token là stateless | `auth.service.ts` |
| Vận chuyển | Header `Authorization: Bearer <access>`; hoặc cookie `accessToken` (httpOnly, maxAge 15 phút, do identity set lúc login) | `*/middleware/auth.middleware.ts` |
| Kiểm tra | **Mỗi service tự verify** JWT bằng `JWT_SECRET` dùng chung: identity, incident, notification, reward (`authenticate`), ai-service (`get_auth_context`). Gateway không kiểm tra gì | `ID/middleware/auth.middleware.ts`, `INC/middleware/auth.middleware.ts`, `NS/middleware/auth.middleware.ts`, `RW/middleware/auth.middleware.ts`, `AI/auth.py` |
| Những gì **không** được kiểm tra | User bị ban, bị xoá, tài khoản đang PENDING_ACTIVATION; access token sau khi logout | như trên |
| Refresh | `POST /api/v1/auth/refresh-token` với cookie `refreshToken` hoặc `body.refreshToken`. Có rotation (revoke token cũ) | `auth.service.ts > refreshAccessToken()` |
| Thu hồi | Logout, đổi mật khẩu, reset mật khẩu, kích hoạt, ban: revoke **mọi REFRESH**. Access token vẫn còn hiệu lực tới khi hết hạn | `auth.service.ts`, `user.service.ts > adminBanUser()` |
| Phía client | Lưu `accessToken`, `refreshToken`, `user` trong `localStorage["auth_store"]` (Zustand persist); gửi Bearer, **`X-Refresh-Token` trong mọi request** và `Accept-Language`; 401 thì refresh 1 lần rồi logout | `FE/stores/useAuthStore.ts`, `FE/libs/axiosClient.ts` |
| Google OAuth | Authorization code flow; không kiểm tra `state`; callback trả JSON token | `ID/modules/oauth/*` |

### 1.2 API key nội bộ (service-to-service)

| Header | Biến | Bảo vệ | Service gọi |
|---|---|---|---|
| `x-internal-api-key` | `INTERNAL_IDENTITY_API_KEY` | identity `/internal/v1/*` | incident, notification, reward (dùng chung 1 key) |
| `x-internal-api-key` | `INTERNAL_INCIDENT_API_KEY` | incident `POST /api/v1/organizations` | Không tìm thấy service nào gọi. Tool `create_organization` của ai-service gọi route này bằng Bearer nên bị 401 |
| `x-internal-api-key` | `INTERNAL_REWARD_API_KEY` | reward `/internal/v1/difficulties*` | incident |
| `x-internal-api-key` | `INTERNAL_NOTIFICATION_API_KEY` | notification `POST /api/v1/notifications/jobs` | incident |
| `x-internal-api-key` | `INTERNAL_AI_API_KEY` | ai `POST /internal/v1/translate` | incident, reward |

- Key được so bằng `!==`, không phải so sánh constant-time.
- Route `/internal/*` không được gateway proxy. Riêng `POST /api/v1/notifications/jobs` và `POST /api/v1/organizations` **có** đi qua gateway, nhưng vẫn cần key.

### 1.3 Token dùng cho luồng đơn đăng ký tổ chức (không có tài khoản)

| Token | Cách có | TTL | Dùng cho | Bằng chứng |
|---|---|---|---|---|
| OTP (6 số) | Email | 10 phút, tối đa 5 lần sai | Mở đơn DRAFT và lấy tracking token | `organization-application-otp.service.ts` |
| LINK token | Link trong email OTP | 10 phút | Xác định email trên form | `resolveEmailLink()` |
| Tracking token (`?token=`) | Xác thực OTP thành công; kèm trong các email gửi người nộp | 180 ngày, dùng lại được | Xem, lưu nháp, tải giấy tờ, nộp, gửi lại lời mời, rút **mọi đơn có cùng `submitterEmail`** | `resolveTrackingToken()`, `loadForApplicant()` |
| Token xác nhận owner (path `/owner-confirmations/:token`) | Email `ORG_OWNER_CONFIRMATION_REQUEST` tới từng owner (đơn NEW_ORG, owner change ADD_OWNER và người nhận TRANSFER_OWNER) | 14 ngày; gửi lại thì token cũ mất hiệu lực | Xem tóm tắt đơn, xác nhận, từ chối — **sở hữu hộp thư là bằng chứng đồng thuận**; lưu sha256 | `owner-confirmation.service.ts` |
| Token lời mời thành viên (path `/api/v1/organization-invitations/:token`) | Email `ORG_INVITATION` khi lời mời được duyệt (SENT) | 7 ngày (`ORG_INVITATION_TTL_DAYS`); lưu sha256 | Xem tóm tắt, chấp nhận (thành MEMBER), từ chối — không cần đăng nhập; JWT (nếu có) chỉ để cảnh báo `session_mismatch` | `INC/modules/organization/organization-invitation.service.ts` |

### 1.4 Token một lần khác

| Token | TTL | Endpoint |
|---|---|---|
| PASSWORD_RESET | 1h | `POST /auth/reset-password` |
| ACCOUNT_ACTIVATION | 72h (`ACCOUNT_ACTIVATION_TTL_MS`), user phải đang PENDING_ACTIVATION | `POST /auth/activate-account` (gửi lại: `POST /auth/activation/resend`) |
| ORGANIZATION_CONTACT_EMAIL | 72h | `GET /organizations/verify-contact-email` |
| QR điểm danh (JWT `campaign_attendance_qr_v1`, ký bằng `JWT_SECRET`) | 1h | `POST /campaigns/:id/attendance-check-in` |

### 1.5 Vai trò

- **Role ở identity:** `ADMIN`, `USER` (role `ORG_OWNER` và tài khoản `accountType = ORG` đã bị xoá bởi migration `20260926100000_drop_org_accounts`). Hệ thống permission set tồn tại nhưng **không được dùng để phân quyền** (`ID/middleware/authorize.middleware.ts` không được route nào gọi).
- **Kiểm tra admin:** tất cả đều là `req.user.role.toLowerCase() === "admin"`, viết lặp lại ở từng controller hoặc middleware (identity `requireAdmin`, reward `requireAdmin`, incident inline trong từng controller).
- **Quyền theo ngữ cảnh** (lưu ở incident): vai trong tổ chức (`organization_members.role`: `LEGAL_REPRESENTATIVE` và `OWNER` là **owner**, ngoài ra `ADMIN`, `CAMPAIGN_MANAGER`, `MEMBER` — gán bằng "đổi vai", xem §2.3). Quyền quản lý tổ chức suy ra từ vai theo **ma trận duy nhất** `DC/org-permissions.ts` và được kiểm tra qua `INC/modules/organization/org-access.service.ts > assertOrgPermission()` (đọc DB mỗi request, không có ngữ cảnh tổ chức trong JWT). Quyền campaign vẫn theo `campaign.createdBy`, manager (`campaign_managers`), owner tổ chức khi tạo — **chưa đổi cho tới phase 4**. Ngoài ra: volunteer APPROVED, chủ report, người được giao task.
- **Phía client:** link Admin chỉ hiện khi `user.roleId === ADMIN_ROLE_ID` (UUID cứng trong `FE/constants/roles.ts`). Guard trong `AdminLayout` **bị comment out**, nên mọi user đăng nhập đều vào được `/admin/*`. Việc chặn thực sự nằm ở API.

## 2. Ma trận phân quyền

Ký hiệu:
- ✅ được
- ❌ không được
- ⚠️ có điều kiện (xem cột ghi chú)
- 🔓 = được nhưng **không đúng thiết kế** (lỗ hổng, xem 99)

Cột:
- **Khách**: không đăng nhập
- **User**: người dùng đăng nhập bất kỳ (owner tổ chức cũng là user thường)
- **Chủ**: chủ tài nguyên theo ngữ cảnh ở cột "Chủ là ai"
- **Admin**

### 2.1 Tài khoản (identity)

| Hành động | Khách | User | Chủ | Admin | Chủ là ai / điều kiện | Nơi kiểm tra |
|---|---|---|---|---|---|---|
| Đăng ký, đăng nhập, Google, refresh, quên và đặt lại mật khẩu, kích hoạt tài khoản, gửi lại email kích hoạt | ✅ | ✅ | | ✅ | | Không có guard |
| Tự chọn role khi đăng ký (`roleId`) | 🔓 | | | | Có thể tự đăng ký làm ADMIN | `auth.service.ts > signup()` |
| Xem `me`, đổi mật khẩu, logout | ❌ | ✅ | | ✅ | | `authenticate` |
| Xem user khác (`GET /users/:id`, `/users/email/:email`) | ❌ | 🔓 | ✅ | ✅ | Lộ email, status, lý do ban. Vị trí chỉ trả cho chính chủ | `user.entity.ts > toUserResponse()` |
| Sửa user (`PUT /users/:id`), kể cả `roleId` | ❌ | 🔓 | ✅ | ✅ | Không kiểm tra ownership | `user.controller.ts > updateUser` |
| Xoá user (`DELETE /users/:id`) | ❌ | 🔓 | ✅ | ✅ | Không kiểm tra ownership | `user.service.ts > deleteUser()` |
| Danh sách user | ❌ | ❌ | | ✅ | | `requireAdmin()` |
| Ban user | ❌ | ❌ | | ⚠️ | Không được tự ban mình | `requireAdmin()`, `adminBanUser()` |
| CRUD role và permission set (`/api/v1/roles/*`) | 🔓 | 🔓 | | ✅ | `authenticate` bị comment out | `ID/modules/role/role.routes.ts` |

### 2.2 Đơn đăng ký tổ chức

| Hành động | Khách | User | Admin | Điều kiện | Nơi kiểm tra |
|---|---|---|---|---|---|
| Xin OTP, xác thực OTP | ✅ | ✅ | ✅ | Rate limit | `rate-limit.middleware.ts` |
| Xem, lưu nháp, tải giấy tờ, nộp, gửi lại lời mời, rút | ⚠️ | ⚠️ | ⚠️ | Có tracking token của email đó; lưu / tải / nộp chỉ khi DRAFT hoặc NEEDS_REVISION | `loadForApplicant()` |
| Xem tóm tắt, xác nhận, từ chối làm owner | ⚠️ | ⚠️ | ⚠️ | Có token xác nhận; JWT (nếu có) chỉ để cảnh báo lệch email | `owner-confirmation.service.ts`, `optionalAuthenticate()` |
| Danh sách và chi tiết đơn, xem giấy tờ | ❌ | ❌ | ✅ | Chỉ đơn `NEW_ORG`; không bao giờ thấy DRAFT / AWAITING_OWNER_CONFIRMATION hay owner change (BR-349) | `organization-application-admin.controller.ts > requireAdmin()`, `HIDDEN_FROM_ADMIN_STATUSES` |
| Claim | ❌ | ❌ | ⚠️ | Chưa bị admin khác claim | `claim()` |
| Yêu cầu bổ sung, duyệt, từ chối, cấp Blue Tick | ❌ | ❌ | ✅ | Chỉ khi PENDING_REVIEW; không yêu cầu phải claim trước | `requestMoreInfo()`, `decide()`, `loadPendingReview()` |

### 2.3 Tổ chức

**Ma trận quyền theo vai** (`DC/org-permissions.ts > ROLE_PERMISSIONS`, `hasOrgPermission()`):

| Quyền (`OrgPermission`) | LEGAL_REPRESENTATIVE / OWNER | ADMIN | CAMPAIGN_MANAGER | MEMBER |
|---|---|---|---|---|
| `ORG_EDIT` — sửa hồ sơ, gửi lại email xác minh | ✅ | ✅ | ❌ | ❌ |
| `MEMBER_APPROVE` — xem và xử lý yêu cầu gia nhập, duyệt / từ chối lời mời | ✅ | ✅ | ❌ | ❌ |
| `MEMBER_INVITE` — mời người có tài khoản làm MEMBER, tìm user | ✅ | ✅ | ✅ | ✅ |
| `MEMBER_MANAGE` — đổi vai, gỡ thành viên | ✅ | ⚠️¹ | ❌ | ❌ |
| `OWNER_PROPOSE` — owner change: thêm owner, thu hồi owner khác, chuyển giao vai của mình; xem / đồng ý / từ chối owner change | ✅ | ❌ | ❌ | ❌ |
| `CAMPAIGN_CREATE`, `CAMPAIGN_MANAGE_ANY` | ✅ | ✅ | chỉ `CAMPAIGN_CREATE` | ❌ |

¹ `assignableRoles(actor)`: owner gán được `ADMIN`, `CAMPAIGN_MANAGER`, `MEMBER`; admin chỉ gán được `CAMPAIGN_MANAGER`, `MEMBER`. `canActOnMember(actor, target)`: không ai đổi vai / gỡ owner hoặc LR qua đây (vai owner chỉ đổi qua owner change được các owner khác đồng ý, hoặc owner tự rút lui), admin không tác động admin khác, không tự tác động chính mình (`INC/modules/organization/organization.service.ts > changeMemberRole(), removeMember()`).

**Owner change** (BR-339..BR-348): quyết trong tổ chức, không qua admin nền tảng.
- ADD_OWNER: từng người được đề xuất xác nhận qua email **và** mọi owner khác (trừ người đề xuất) đồng ý.
- REMOVE_OWNER: mọi owner trừ người đề xuất và người bị thu hồi đồng ý; tổ chức có 2 owner thì có hiệu lực ngay. Người bị thu hồi không phủ quyết được.
- TRANSFER_OWNER: chỉ cho thành viên hiện tại không phải owner; người nhận chấp nhận qua email là đủ, owner khác chỉ được báo.
- Owner tự hạ vai / rời: có hiệu lực ngay khi còn owner khác.

`CAMPAIGN_CREATE` / `CAMPAIGN_MANAGE_ANY` mới **khai báo** trong ma trận và trả trong `permissions`; `createCampaign()` vẫn chỉ cho owner và quyền quản lý campaign chưa theo vai — **[CHƯA HOÀN THIỆN] phase 4**.

Mọi response tổ chức có người xem (`GET /:id`, `/by-slug/:slug`, `/`, `/my`) kèm `my_role`, `is_owner`, `is_member` và `permissions { can_edit_org, can_approve_members, can_invite, can_manage_members, can_propose_owners, can_create_campaign, can_manage_all_campaigns, assignable_roles }` (`permissionsFor(role)`); client ẩn / hiện nút theo object này.

| Hành động | Khách | User | Owner | Admin tổ chức | CM / Member | Admin nền tảng | Nơi kiểm tra |
|---|---|---|---|---|---|---|---|
| Tạo tổ chức trực tiếp | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ (chỉ service có `INTERNAL_INCIDENT_API_KEY`) | `requireInternalIncidentApiKey()` |
| Xem danh sách, chi tiết, theo slug | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | `authenticate` |
| Sửa thông tin, gửi lại email xác minh | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | `ORG_EDIT` |
| Xác minh email liên hệ (click link) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Token |
| Duyệt hoặc ban tổ chức | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | `adminVerifyOrganization` |
| Xin gia nhập | ❌ | ⚠️ (chưa là thành viên, chưa có PENDING) | ❌ | ❌ | ❌ | ⚠️ | `createJoinRequest()` |
| Xem và xử lý yêu cầu gia nhập | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | `MEMBER_APPROVE` |
| Huỷ yêu cầu gia nhập của mình | ❌ | ⚠️ (khi PENDING) | | | | | `cancelJoinRequest()` |
| Tìm user để mời (`GET /:id/user-search`) | ❌ | ❌ | ✅ (email đầy đủ) | ✅ (email ẩn) | ✅ (email ẩn) | ❌ | `MEMBER_INVITE`; email đầy đủ chỉ khi có `OWNER_PROPOSE` |
| Tạo lời mời MEMBER | ❌ | ❌ | ✅ (gửi ngay) | ✅ (gửi ngay) | ✅ (chờ duyệt) | ❌ | `MEMBER_INVITE`; có `MEMBER_APPROVE` thì SENT ngay |
| Xem lời mời | ❌ | ❌ | ✅ tất cả | ✅ tất cả | ⚠️ chỉ lời mời của mình | ❌ | `organization-invitation.service.ts > list()` |
| Duyệt / từ chối lời mời | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | `MEMBER_APPROVE` |
| Huỷ lời mời đang mở | ❌ | ❌ | ✅ | ✅ | ⚠️ lời mời của mình | ❌ | `cancel()` |
| Chấp nhận / từ chối lời mời (theo token) | ⚠️ | ⚠️ | | | | | Token lời mời (§1.3) |
| Đổi vai, gỡ thành viên | ❌ | ❌ | ✅ (trừ owner) | ⚠️¹ | ❌ | ❌ | `MEMBER_MANAGE` + `canActOnMember()` |
| Tạo owner change (ADD_OWNER / REMOVE_OWNER / TRANSFER_OWNER), xem danh sách, gửi lại lời xác nhận | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | `OWNER_PROPOSE` (`owner-change.service.ts`) |
| Đồng ý / từ chối owner change | ❌ | ❌ | ⚠️ chỉ owner được hỏi (approver PENDING) | ❌ | ❌ | ❌ | `owner-change.service.ts > answer()` (BR-339, BR-347) |
| Huỷ owner change | ❌ | ❌ | ⚠️ chỉ người đề xuất | ❌ | ❌ | ❌ | `cancel()` |
| Chấp nhận nhận thêm / nhận chuyển giao vai owner (theo token) | ⚠️ | ⚠️ | | | ⚠️ người nhận chuyển giao là member | | Token xác nhận owner (§1.3) |
| Duyệt owner change | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ (admin nền tảng không còn duyệt ADD_OWNER; can thiệp trực tiếp vai owner là việc sau) | BR-349 |
| Tự hạ vai (`PATCH /:id/members/me/role`) | ❌ | ❌ | ⚠️ còn owner khác | ❌ | ❌ | | `stepDown()` |
| Rời tổ chức | ❌ | | ⚠️ còn owner khác (owner cuối: 409 `ORG_MUST_HAVE_OWNER`) | ✅ | ✅ | | `leaveOrganization()`, `ownerStepOut()` |
| Xem danh sách thành viên (kèm `role`) | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | Không có guard — cần xác nhận có chủ ý công khai hay không (`listMembersForOwner()`) |

### 2.4 Báo cáo sự cố

| Hành động | Khách | User | Chủ report | Admin | Nơi kiểm tra |
|---|---|---|---|---|---|
| Tạo report | ❌ | ✅ | | ✅ | `authenticate` |
| Tìm kiếm, xem danh sách, xem chi tiết | ❌ | ✅ (kể cả report PENDING của người khác qua `/search`, `/by-ids`) | ✅ | ✅ | `authenticate`; report bị ban trả 404 ở chi tiết |
| Xem ảnh theo ids | ❌ | ⚠️ (report đã duyệt) | ✅ | ⚠️ | `findManyByIdsVisibleToViewer()` |
| Sửa, thêm hoặc xoá ảnh, xoá report | ❌ | ❌ | ⚠️ (report chưa bị ban) | ❌ (bị cấm tường minh) | `assertReporterMayEditReport()` |
| Duyệt, ban, đánh dấu đã xử lý | ❌ | ❌ | ❌ | ✅ | `report.controller.ts` |
| Xem trạng thái job AI | ❌ | 🔓 | ✅ | ✅ | Không kiểm tra chủ |
| Vote, lưu | ❌ | ✅ | ✅ (tự vote được) | ✅ | `authenticate` |
| Đăng ký media catalog | ❌ | ❌ | | ✅ | `admin-media.controller.ts > assertAdmin()` |

### 2.5 Chiến dịch

Cột **Owner tổ chức** (membership vai `LEGAL_REPRESENTATIVE` / `OWNER`) dùng cho hành động tạo campaign. Cột **createdBy** là người đã tạo campaign. Cột **Manager** là người có trong `campaign_managers` (người tạo tự động được thêm vào bảng này). Cột **Volunteer** là người có join request APPROVED.

| Hành động | User | Owner tổ chức | createdBy | Manager | Volunteer | Admin | Nơi kiểm tra |
|---|---|---|---|---|---|---|---|
| Tạo campaign | ❌ | ✅ | | | | ❌ (trừ khi là owner) | `createCampaign()`, `isOwner()` |
| Xem danh sách, chi tiết (mọi status), task, manager, submission | 🔓 ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | `authenticate` |
| Sửa campaign (kể cả `status`, report, manager) | ❌ | | ✅ 🔓 (đổi được status) | ❌ | ❌ | ❌ | `ensureOwner()` |
| Xoá campaign | ❌ | | ✅ | ❌ | ❌ | ❌ | `ensureOwner()` |
| Duyệt hoặc ban campaign | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | `campaign.controller.ts > adminVerifyCampaign` |
| Thêm hoặc gỡ manager | ❌ | | ✅ | ✅ | ❌ | ❌ | `canManageCampaign()` |
| Tạo, sửa, xoá, giao task | ❌ | | ✅ | ✅ | ❌ | ❌ | `canManageCampaign()` |
| Cập nhật kết quả task | ❌ | | ✅ | ✅ | ⚠️ (task được giao) | ❌ | `updateTaskResult()` |
| Đổi status task qua `/status` | ❌ | | | | ⚠️ (task được giao) | ❌ | `updateTaskStatusByVolunteer()` |
| Xin tham gia | ✅ | ✅ | ✅ (tự xin được) | ✅ | | ✅ | `createJoinRequest()` |
| Xem và xử lý yêu cầu tham gia | ❌ | | ⚠️ (chỉ khi còn trong bảng manager) | ✅ | ❌ | ❌ | `isManager` |
| Xem danh sách volunteer đã duyệt | 🔓 | ✅ | ✅ | ✅ | ✅ | ✅ | Kiểm tra bị comment out |
| Tạo QR điểm danh | ❌ | | ✅ | ✅ | ❌ | ❌ | `canManageCampaign()` |
| Check-in | ❌ | | | | ✅ | | `checkInWithQrToken()` |
| Gửi hoàn thành (mark-done) | ❌ | | ⚠️ (chỉ khi còn trong bảng manager) | ✅ | ❌ | ❌ | `isManager` |
| Xác nhận sạch (completion-verification) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | `authenticate` |
| Duyệt hoặc từ chối hoàn thành | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | `adminReviewCampaignCompletion` |
| Tạo và duyệt submission | ❌ | | ⚠️ | ✅ (tự duyệt được) | ❌ | ❌ | `isManager` |
| Danh sách "multi-submission review" | ❌ | | | | | ✅ | controller |
| Gửi SOS, xem SOS | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | `authenticate` |
| Giải quyết SOS | 🔓 | 🔓 | 🔓 | ✅ | 🔓 | ✅ | Không kiểm tra quyền |

### 2.6 Điểm, quà tặng, gamification (reward)

| Hành động | Khách | User | Admin | Nơi kiểm tra |
|---|---|---|---|---|
| Xem quà, chi tiết quà, difficulty, leaderboard, season hiện tại, ước tính thưởng, bảng xếp hạng gamification | ✅ | ✅ | ✅ | Không có guard |
| Lọc quà theo `isActive` | ❌ | ❌ | ✅ | `gift.api.routes.ts` (tryAuthenticate) |
| Đổi quà; xem điểm, lịch sử, đơn, badge, tổng hợp của mình | ❌ | ✅ | ✅ | `authenticate` |
| Xem metric tables và columns | ❌ | ✅ | ✅ | `authenticate` |
| Tạo và sửa quà; sửa difficulty; xem mọi đơn đổi quà | ❌ | ❌ | ✅ | `requireAdmin` |
| **Đổi trạng thái đơn đổi quà** | ❌ | 🔓 | ✅ | Thiếu `requireAdmin` (`gift.api.routes.ts`) |
| Mọi `/admin/gamification/*`, `/admin/seasons/*` | ❌ | ❌ | ✅ | `requireAdmin` |
| `/internal/v1/difficulties*` | Chỉ service có `INTERNAL_REWARD_API_KEY` | | | `requireInternalRewardApiKey` |

### 2.7 Thông báo

| Hành động | Khách | User | Admin | Service nội bộ | Nơi kiểm tra |
|---|---|---|---|---|---|
| Gửi thông báo (enqueue job) | ❌ | ❌ | ❌ | ✅ | `NS/middleware/internal-auth.middleware.ts` |
| Xem thông báo của mình, đánh dấu đã đọc | ❌ | ✅ (chỉ của mình) | ✅ | | `authenticate` + lọc theo userId |

Không có API admin cho thông báo, và không có cách xem lịch sử email qua API.

### 2.8 AI

| Hành động | Khách | User | Nơi kiểm tra |
|---|---|---|---|
| Xem danh sách agent | ✅ | ✅ | Không có guard |
| Tạo hội thoại, gửi tin nhắn, đăng ký media, dịch | ❌ | ✅ | `get_auth_context` |
| Đọc hoặc stream hội thoại | ❌ | ⚠️ (chỉ của mình) | `get_conversation_for_user()` |
| Tool call sang incident | | Theo quyền của chính user ở incident (dùng Bearer của user) | `AI/tools/*` |
| `POST /api/v1/recommendations/report`, `/api/v1/social/campaign-facebook-caption` | Không có kiểm tra. Chỉ an toàn nhờ gateway không proxy | | `AI/recommendation/router.py`, `AI/social/router.py` |
| `/internal/v1/translate` | Service có `INTERNAL_AI_API_KEY` | | `require_internal_api_key()` |

## 3. Tóm tắt các chỗ thiếu kiểm tra quyền

Chi tiết từng vấn đề ở [99-open-issues.md](99-open-issues.md) mục A.

1. Sign-up nhận `roleId` từ client, nên ai cũng có thể tự nâng quyền lên admin.
2. `PUT` và `DELETE /users/:id` không kiểm tra chủ sở hữu.
3. `/api/v1/roles/*` không yêu cầu đăng nhập.
4. `request-password-reset` trả token reset trong response.
5. `PATCH /admin/gift-redemptions/:id/status` thiếu kiểm tra admin.
6. `PUT /campaigns/:id` cho người tạo đổi `status` tuỳ ý.
7. Danh sách volunteer đã duyệt, giải quyết SOS và trạng thái job AI không kiểm tra quyền (campaign / SOS để phase 4). Danh sách thành viên tổ chức cũng không có guard — cần xác nhận.
8. Guard `/admin` phía client bị comment out.
9. ~~Refresh token đổi claim `role` thành UUID~~ — đã sửa (`refreshAccessToken()` dùng tên role).
