# 05 — Xác thực và phân quyền

> Đường dẫn viết tắt giống [02-business-flows.md](02-business-flows.md).

## 1. Cơ chế xác thực

### 1.1 JWT (người dùng)

| Mục | Hành vi trong code | Bằng chứng |
|---|---|---|
| Phát hành | identity ký **access token** và **refresh token** bằng cùng `JWT_SECRET`, cùng payload `{userId, email, role}`, không có claim phân biệt loại token. TTL: `JWT_EXPIRES_IN` (code mặc định 30m) và `JWT_REFRESH_EXPIRES_IN` (30d) | `ID/utils/jwt.utils.ts`, `ID/modules/auth/auth.service.ts > login()` |
| Claim `role` | Khi login: **tên role** (`ADMIN`, `USER`, `ORG_OWNER`). Khi refresh: **roleId (UUID)**. Đây là bug làm admin mất quyền sau khi refresh | `auth.service.ts` (khoảng dòng 137 và 189) |
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
| OTP (6 số) | Email | 10 phút, tối đa 5 lần sai | Đổi lấy submission token | `organization-application-otp.service.ts` |
| LINK token | Link trong email OTP | 10 phút | Xác định email trên form | `resolveEmailLink()` |
| Submission token (`x-submission-token`) | Xác thực OTP thành công | 30 phút, dùng 1 lần | Presign giấy tờ, nộp đơn | `submission-token.middleware.ts` |
| Tracking token (`?token=`) | Sau khi nộp đơn hoặc khi được yêu cầu bổ sung | 180 ngày | Xem, sửa, rút **mọi đơn cùng email** | `resolveTrackingToken()` |

### 1.4 Token một lần khác

| Token | TTL | Endpoint |
|---|---|---|
| PASSWORD_RESET | 1h | `POST /auth/reset-password` |
| ORG_ACCOUNT_ACTIVATION | 72h | `POST /auth/activate-org-account` |
| ORGANIZATION_CONTACT_EMAIL | 72h | `GET /organizations/verify-contact-email` |
| QR điểm danh (JWT `campaign_attendance_qr_v1`, ký bằng `JWT_SECRET`) | 1h | `POST /campaigns/:id/attendance-check-in` |

### 1.5 Vai trò

- **Role ở identity:** `ADMIN`, `USER`, `ORG_OWNER`. Hệ thống permission set tồn tại nhưng **không được dùng để phân quyền** (`ID/middleware/authorize.middleware.ts` không được route nào gọi).
- **Kiểm tra admin:** tất cả đều là `req.user.role.toLowerCase() === "admin"`, viết lặp lại ở từng controller hoặc middleware (identity `requireAdmin`, reward `requireAdmin`, incident inline trong từng controller).
- **Quyền theo ngữ cảnh** (lưu ở incident): owner tổ chức (`organizations.ownerId`), thành viên tổ chức, `campaign.createdBy`, manager (`campaign_managers`), volunteer APPROVED, chủ report, người được giao task.
- **Phía client:** link Admin chỉ hiện khi `user.roleId === ADMIN_ROLE_ID` (UUID cứng trong `FE/constants/roles.ts`). Guard trong `AdminLayout` **bị comment out**, nên mọi user đăng nhập đều vào được `/admin/*`. Việc chặn thực sự nằm ở API.

## 2. Ma trận phân quyền

Ký hiệu:
- ✅ được
- ❌ không được
- ⚠️ có điều kiện (xem cột ghi chú)
- 🔓 = được nhưng **không đúng thiết kế** (lỗ hổng, xem 99)

Cột:
- **Khách**: không đăng nhập
- **User**: người dùng đăng nhập bất kỳ, kể cả tài khoản tổ chức
- **Chủ**: chủ tài nguyên theo ngữ cảnh ở cột "Chủ là ai"
- **Admin**

### 2.1 Tài khoản (identity)

| Hành động | Khách | User | Chủ | Admin | Chủ là ai / điều kiện | Nơi kiểm tra |
|---|---|---|---|---|---|---|
| Đăng ký, đăng nhập, Google, refresh, quên và đặt lại mật khẩu, kích hoạt tài khoản tổ chức | ✅ | ✅ | | ✅ | | Không có guard |
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
| Presign giấy tờ, nộp đơn | ⚠️ | ⚠️ | ⚠️ | Có submission token | `requireSubmissionToken()` |
| Xem, sửa, rút đơn | ⚠️ | ⚠️ | ⚠️ | Có tracking token của email đó; chỉ sửa được khi NEEDS_MORE_INFO | `loadForApplicant()` |
| Danh sách và chi tiết đơn, xem giấy tờ | ❌ | ❌ | ✅ | | `organization-application-admin.controller.ts > requireAdmin()` |
| Claim | ❌ | ❌ | ⚠️ | Chưa bị admin khác claim | `claim()` |
| Yêu cầu bổ sung, duyệt, từ chối, cấp Blue Tick | ❌ | ❌ | ✅ | Không yêu cầu phải claim trước | `requestMoreInfo()`, `decide()` |

### 2.3 Tổ chức

| Hành động | Khách | User | Owner | Thành viên | Admin | Nơi kiểm tra |
|---|---|---|---|---|---|---|
| Tạo tổ chức trực tiếp | ❌ | ❌ | ❌ | ❌ | ❌ (chỉ service có `INTERNAL_INCIDENT_API_KEY`) | `requireInternalIncidentApiKey()` |
| Xem danh sách, chi tiết, theo slug | ❌ | ✅ | ✅ | ✅ | ✅ | `authenticate` |
| Sửa thông tin tổ chức | ❌ | ❌ | ✅ | ❌ | ❌ | `updateOrganization()` |
| Gửi lại email xác minh | ❌ | ❌ | ✅ | ❌ | ❌ | `resendOrganizationContactVerificationEmail()` |
| Xác minh email liên hệ (click link) | ✅ | ✅ | ✅ | ✅ | ✅ | Token |
| Duyệt hoặc ban tổ chức | ❌ | ❌ | ❌ | ❌ | ✅ | `adminVerifyOrganization` |
| Xin gia nhập | ❌ | ⚠️ (chưa là thành viên, chưa có PENDING) | ❌ | ❌ | ⚠️ | `createJoinRequest()` |
| Xem và xử lý yêu cầu gia nhập | ❌ | ❌ | ✅ | ❌ | ❌ | `processJoinRequest()` |
| Huỷ yêu cầu gia nhập của mình | ❌ | ⚠️ (khi PENDING) | | | | `cancelJoinRequest()` |
| Rời tổ chức | ❌ | | ❌ | ✅ | | `leaveOrganization()` |
| Xem danh sách thành viên | ❌ | 🔓 | ✅ | 🔓 | ✅ | Kiểm tra owner bị comment out (`listMembersForOwner()`) |

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

Cột **Owner tổ chức** dùng cho hành động tạo campaign. Cột **createdBy** là người đã tạo campaign. Cột **Manager** là người có trong `campaign_managers` (người tạo tự động được thêm vào bảng này). Cột **Volunteer** là người có join request APPROVED.

| Hành động | User | Owner tổ chức | createdBy | Manager | Volunteer | Admin | Nơi kiểm tra |
|---|---|---|---|---|---|---|---|
| Tạo campaign | ❌ | ✅ | | | | ❌ (trừ khi là owner) | `createCampaign()` |
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
7. Xem danh sách thành viên tổ chức, danh sách volunteer đã duyệt, giải quyết SOS và trạng thái job AI đều không kiểm tra quyền.
8. Guard `/admin` phía client bị comment out.
9. Refresh token đổi claim `role` thành UUID, làm admin mất quyền (theo hướng an toàn, nhưng là bug).
