# 03 — Business rules

> Tất cả rule trong bảng đều tìm thấy trong code. Viết tắt đường dẫn giống [02-business-flows.md](02-business-flows.md): `ID/`, `INC/`, `RW/`, `NS/`, `AI/`, `FE/`, `DC/` (= `ecolink-server/shared/da2-constants/src`).
> Dải mã theo module: 001–019 xác thực · 020–049 người dùng và nội bộ identity · 050–069 nộp đơn tổ chức · 070–079 thẩm định đơn · 300–329 owner xác nhận và membership tổ chức · 080–099 tổ chức · 100–129 report · 130–149 vote và lưu · 150–189 campaign · 190–199 SOS · 200–219 thông báo · 220–239 quà tặng · 240–269 điểm và gamification · 270–289 AI và dịch · 290–299 kiểm tra phía client. Một số mã trong dải được để trống để dành chỗ.
> Mã lỗi theo envelope `{success:false, code, message}` (`DC/http-status.ts > sendError`).

## 1. Xác thực (identity và các service)

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-001 | Sign-up: email hợp lệ, password ≥ 8, name không rỗng, roleId (tuỳ chọn) là UUID | `POST /api/v1/auth/sign-up` | 400 VALIDATION_ERROR | `ID/modules/auth/auth.controller.ts > signup` |
| BR-002 | Email không được trùng với user chưa bị xoá (so khớp chính xác, phân biệt hoa thường) | sign-up | 409 "User with this email already exists" | `ID/modules/auth/auth.service.ts > signup()` |
| BR-003 | Không gửi roleId thì gán role `USER`; có gửi thì dùng roleId đó (**client tự chọn role được**) | sign-up | Thiếu role USER → 500 | `auth.service.ts > signup()` |
| BR-004 | Sign-in: sai email hoặc mật khẩu, hoặc tài khoản không có mật khẩu | `POST /auth/sign-in` | 401 INVALID_CREDENTIALS | `auth.service.ts > login()` |
| BR-005 | Tài khoản status=2 (bị ban) hoặc status=3 (`PENDING_ACTIVATION`: tài khoản tạo cho owner được duyệt, chưa đặt mật khẩu) không đăng nhập được, kể cả qua Google | sign-in, Google callback | 403 "Account banned" / 403 ACCOUNT_PENDING_ACTIVATION | `auth.service.ts > login()`, `ID/modules/oauth/google.service.ts > handleCallback()`, `ID/modules/auth/auth.controller.ts` |
| BR-006 | Access token lấy từ header `Authorization: Bearer` hoặc cookie `accessToken`, verify bằng `JWT_SECRET` | Mọi route có `authenticate` ở identity, incident, notification, reward; ai-service `get_auth_context` | 401 TOKEN_MISSING / TOKEN_INVALID | `*/middleware/auth.middleware.ts > authenticate()`, `AI/auth.py` |
| BR-007 | `authenticate` **không** kiểm tra user có bị ban hoặc bị xoá hay không. Token còn hạn thì vẫn dùng được | toàn hệ thống | — | như trên |
| BR-008 | Refresh token phải: verify được, có bản ghi REFRESH chưa revoke/dùng/hết hạn, userId khớp, user chưa bị xoá. Mỗi lần refresh sẽ revoke token cũ (rotation). Access token mới mang **tên** role (như lúc login) | `POST /auth/refresh-token` | 401 "Invalid refresh token" | `auth.service.ts > refreshAccessToken()` |
| BR-009 | Đổi mật khẩu cần đúng mật khẩu cũ; mật khẩu mới ≥ 8; đổi xong thì revoke mọi REFRESH | `POST /auth/update-password` | 400 "Invalid old password" | `auth.service.ts > updatePassword()` |
| BR-010 | Token reset mật khẩu dùng 1 lần, TTL `PASSWORD_RESET_TTL_MS` (1h). Phát token mới thì token cũ bị thu hồi | request-password-reset, reset-password | 404 email không tồn tại; 400 "Invalid or expired reset token" | `auth.service.ts > requestPasswordReset(), resetPassword()` |
| BR-011 | Kích hoạt tài khoản: token `ACCOUNT_ACTIVATION` còn hiệu lực (TTL `ACCOUNT_ACTIVATION_TTL_MS`, mặc định 72h), user đang `PENDING_ACTIVATION`, password ≥ 8. Kích hoạt xong thì status=1 và revoke mọi REFRESH | `POST /auth/activate-account` | 400 "Invalid or expired activation token" | `auth.service.ts > activateAccount()` |
| BR-012 | Logout revoke mọi REFRESH và xoá cookie; access token không bị thu hồi | `POST /auth/logout` | — | `auth.service.ts > logout()` |
| BR-013 | Google callback: có `error` hoặc thiếu `code` thì từ chối; email lần đầu xuất hiện thì tạo user USER | Google callback | 400 / 500 | `ID/modules/auth/auth.controller.ts > googleCallback` |
| BR-014 | Admin = claim `role` của JWT, so với `"admin"` không phân biệt hoa thường | Mọi endpoint admin ở identity, incident, reward | 403 | `ID/modules/user/user.controller.ts > requireAdmin()`, `RW/middleware/require-admin.middleware.ts`, các controller của incident |
| BR-015 | API key nội bộ: header `x-internal-api-key` phải bằng biến env tương ứng | `/internal/v1/*` (identity, reward, ai), `POST /api/v1/notifications/jobs`, `POST /api/v1/organizations` | Chưa cấu hình → 500; sai → 401 "Invalid internal API key" | `*/middleware/internal-*.middleware.ts`, `AI/auth.py > require_internal_api_key()` |
| BR-016 | Gửi lại email kích hoạt: luôn trả 200 (không lộ email nào tồn tại); chỉ phát token khi user đang `PENDING_ACTIVATION` và chưa có ≥ 3 token `ACCOUNT_ACTIVATION` trong 60 phút; email đi qua notification-service (thiếu `NOTIFICATION_SERVICE_URL` / `INTERNAL_NOTIFICATION_API_KEY` thì chỉ log cảnh báo) | `POST /auth/activation/resend` | — | `auth.service.ts > requestActivationResend()`, `ID/modules/auth/account-activation-notify.client.ts` |

## 2. Người dùng và identity nội bộ

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-020 | Cập nhật hồ sơ: avatar null hoặc URL http(s); phone ≤ 20 ký tự theo regex `^[0-9+\s\-().]{7,20}$`; gender ∈ male/female/other/prefer_not_to_say; dateOfBirth YYYY-MM-DD; detailAddress ≤ 255; roleId là UUID | `PUT /api/v1/users/:id` | 400 VALIDATION_ERROR | `ID/modules/user/user.controller.ts > updateUser` |
| BR-021 | latitude và longitude phải gửi cùng nhau; cả hai null thì xoá vị trí và xoá luôn detailAddress | như trên | 400 "latitude and longitude must be provided together" | `ID/modules/user/user.service.ts > updateUser()` |
| BR-022 | latitude ∈ [-90, 90], longitude ∈ [-180, 180], phải là kiểu number | như trên | 400 | `user.controller.ts` |
| BR-023 | notificationPreferences được merge với giá trị hiện có, chỉ nhận key hợp lệ có giá trị boolean | như trên | Key sai bị bỏ qua | `user.service.ts > updateUser()`, `DC/notification-preferences.ts > mergeNotificationPreferences()` |
| BR-024 | Thông tin vị trí chỉ được trả khi người xem chính là user đó | `GET /users/:id`, `/users/email/:email` | Field bị ẩn | `ID/modules/user/user.entity.ts > toUserResponse()` |
| BR-025 | Ban: cần reject_reason (trim, ≤ 5000); không được tự ban mình; ban lại với lý do khác thì chỉ cập nhật lý do | `PUT /users/:id/ban` (admin) | 400 "Cannot ban yourself" / reason bắt buộc; 404 | `user.service.ts > adminBanUser()` |
| BR-026 | Ban sẽ đặt status=2 và revoke mọi REFRESH. Không có API gỡ ban | như trên | — | như trên |
| BR-027 | Danh sách user (admin): status ∈ {1, 2}; sort_by ∈ created_at/name/email; limit 1..100 (mặc định 10); search ILIKE trên name và email | `GET /api/v1/users` | 400 | `user.controller.ts > getAllUsers`, `user.repository.ts > findManyForAdmin()` |
| BR-028 | Tên role và tên permission set là duy nhất; permission phải thuộc enum `Permission` | `/api/v1/roles/*` | 409 / 400 | `ID/modules/role/role.service.ts` |
| BR-029 | ensure-users (khi duyệt đơn): mỗi email (so không phân biệt hoa thường) tìm user sẵn có; chưa có thì tạo user role `USER`, `password = null`, status 3, `emailVerified = true`. Idempotent theo email; hai lần tạo đua nhau thì bên thua đọc lại bản ghi của bên thắng. 1..20 user mỗi lần | `POST /internal/v1/users/ensure` | 400; 500 nếu thiếu role USER | `ID/modules/auth/auth.service.ts > ensureUsersForOwners()` |
| BR-030 | Token xác minh email liên hệ tổ chức: 72h, dùng 1 lần, phát mới thì token cũ của cùng tổ chức bị thu hồi | `/internal/v1/organization-contact-email/tokens(/verify)` | 404 "Invalid or expired token" | `auth.service.ts > createOrganizationContactEmailToken()`, `ID/modules/auth/auth_token.repository.ts` |
| BR-031 | nearby-ids: radiusMeters 1..200000 (mặc định 5000), excludeUserIds ≤ 500, tính theo Haversine; không lọc theo status | `POST /internal/v1/users/nearby-ids` | 400 | `ID/modules/user/user.repository.ts > findActiveUserIdsNearPoint()` |
| BR-032 | by-ids: 1..100 UUID, loại trùng, bỏ user đã bị xoá | `POST /internal/v1/users/by-ids` | 400 | `user.repository.ts > findByIds()` |
| BR-033 | Lọc theo preference: userIds 1..500. Kind admin-only hoặc kind không nhận ra thì luôn giữ user. Id không tồn tại vẫn được giữ | `POST /internal/v1/users/notification-prefs/filter` | — | `user.service.ts > filterUserIdsForNotificationKind()`, `DC/notification-preferences.ts` |
| BR-034 | lookup-by-emails: 1..20 email hợp lệ, so không phân biệt hoa thường, bỏ user đã xoá; trả `id, email, name, status, createdAt` | `POST /internal/v1/users/lookup-by-emails` | 400 | `auth.service.ts > lookupUsersByEmails()`, `user.repository.ts > findManyByEmails()` |
| BR-035 | Phát token kích hoạt nội bộ: chỉ khi user đang `PENDING_ACTIVATION` (ngược lại `activation_token = null`); thu hồi token cũ | `POST /internal/v1/users/:id/activation-token` | 400 | `auth.service.ts > issueActivationToken()` |

## 3. Nộp đơn đăng ký tổ chức

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-050 | Rate limit xin OTP: 3 lần/email/giờ (không tính lượt khi request trả 4xx hoặc 5xx), 10 lần/IP/giờ. Các endpoint public khác: 60 lần/IP/giờ. IP lấy từ phần tử đầu của X-Forwarded-For | `/api/v1/organization-applications/*` | 429 TOO_MANY_REQUESTS | `INC/middleware/rate-limit.middleware.ts` |
| BR-051 | Chỉ tắt được rate limit khi `NODE_ENV != production` **và** `APPLICATION_RATE_LIMIT_DISABLED=true` | như trên | — | `rate-limit.middleware.ts > rateLimitDisabled()` |
| BR-052 | Bộ đếm OTP trong DB: quá 3 OTP cho cùng email trong 60 phút thì chặn | `POST /email-otp` | 429 | `INC/modules/organization_application/organization-application-otp.service.ts > requestOtp()` |
| BR-053 | OTP gồm 6 chữ số (CSPRNG), TTL 10 phút, lưu dạng hash; gửi email lỗi thì xoá OTP vừa tạo | `POST /email-otp` | 503 "Could not send the verification code…" | `requestOtp()` |
| BR-054 | Xác thực OTP: tối đa 5 lần sai (quá thì OTP bị huỷ). Đúng thì mở **đơn DRAFT** cho email đó (hoặc trả đơn đang mở, `resumed=true`) và cấp tracking token 180 ngày | `POST /email-otp/verify` | 400 OTP_INVALID; 429 OTP_TOO_MANY_ATTEMPTS | `organization-application-otp.service.ts > verifyOtp()`, `organization-application.service.ts > openDraftForEmail()` |
| BR-055 | Mọi endpoint của người nộp sau OTP dùng tracking token (`?token=`); email của token phải trùng `submitterEmail` của đơn | `PUT /:id`, `POST /:id/submit`, `POST /:id/documents/presign`, `POST /:id/owners/:candidateId/resend`, `POST /:id/withdraw`, `GET /:id` | 401 TRACKING_TOKEN_INVALID; 404 | `resolveTrackingToken()`, `loadForApplicant()` |
| BR-056 | Giấy tờ: docType thuộc `ApplicationDocType`; chỉ nhận pdf/jpeg/png; ≤ 10MB; ≤ 5 file mỗi đơn; chữ ký upload hết hạn sau 15 phút; chỉ khi đơn DRAFT hoặc NEEDS_REVISION | `POST /:id/documents/presign` | 400 INVALID_INPUT; 422 ORGANIZATION_DOCUMENT_LIMIT; 409 | `organization-application.service.ts > presignDocumentForApplication()`, `storage/cloudinary-document-storage.ts` |
| BR-057 | Phải đồng ý xử lý dữ liệu cá nhân trước khi nộp (đã lưu `consentedAt` hoặc gửi `consent=true` khi submit) | `POST /:id/submit` | 400 "Consent to personal data processing is required" | `submitApplication()` |
| BR-058 | orgType ∈ GOV, SCHOOL, CLUB, NGO, SOCIAL_ENTERPRISE; bắt buộc khi nộp, tuỳ chọn khi lưu nháp | `PUT /:id`, `POST /:id/submit` | 400 | `saveDraft()`, `submitApplication()` |
| BR-059 | Khi nộp: profile bắt buộc có name và logoUrl. contactEmail tuỳ chọn, mặc định là `submitterEmail`, phải là email hợp lệ. Toạ độ (nếu có) phải trong khoảng hợp lệ. Lưu nháp chỉ kiểm tra những giá trị có mặt | `PUT /:id`, `POST /:id/submit` | 400 | `sanitizeProfile()`, `validateProfile()` |
| BR-060 | Khi nộp: có ≥ 1 kênh chính thức (FACEBOOK_PAGE / WEBSITE / ZALO_OA), URL bắt đầu bằng http(s):// | `PUT /:id`, `POST /:id/submit` | 400 "At least one official channel is required" | `validateChannels()` |
| BR-061 | Mỗi email người nộp chỉ có **1 đơn đang mở** (DRAFT, AWAITING_OWNER_CONFIRMATION, PENDING_REVIEW, NEEDS_REVISION). OTP lần hai mở lại đơn đó thay vì tạo mới. Không có ràng buộc ở DB | `POST /email-otp/verify` | — | `organization-application.repository.ts > findOpenBySubmitterEmail()` |
| BR-063 | KYC người đại diện pháp lý (của owner có `isLegalRep`): mọi trường tuỳ chọn; idType ∈ CCCD/MSSV/PASSPORT/OTHER; idNumber ≥ 4 ký tự, chỉ lưu hash và 4 ký tự cuối; bỏ trống thì giữ số đã lưu | `PUT /:id` | 400 | `sanitizeLegalRep()` |
| BR-064 | Giấy tờ đính kèm phải tồn tại và do chính `submitterEmail` tải lên; tối đa 5; `nationalIdDocumentId` của owner phải trỏ tới giấy tờ đang đính kèm | `PUT /:id` | 404 ORGANIZATION_DOCUMENT_NOT_FOUND; 422 | `assertDocumentsOwnedBy()`, `saveDraft()` |
| BR-065 | Mã đơn có dạng `ORG-` + 8 ký tự hex viết hoa, thử lại 5 lần nếu trùng; được cấp ngay khi mở DRAFT | `POST /email-otp/verify` | 409 | `createWithUniqueCode()` |
| BR-066 | Tracking token (180 ngày) gắn với email, dùng lại được, cho phép xem, lưu, nộp, gửi lại lời mời và rút mọi đơn có cùng `submitterEmail`. Các email gửi người nộp (nhận đơn, yêu cầu sửa, owner từ chối / hết hạn) mang token mới | `GET/PUT /:id`, các `POST /:id/*` | 401 / 404 | `resolveTrackingToken()`, `loadForApplicant()` |
| BR-067 | Chỉ lưu nháp, tải giấy tờ và nộp khi đơn ở **DRAFT** hoặc **NEEDS_REVISION** (kiểm tra lại dưới row lock) | `PUT /:id`, `POST /:id/submit`, `POST /:id/documents/presign` | 409 ORGANIZATION_APPLICATION_NOT_EDITABLE | `saveDraft()`, `submitApplication()`, `presignDocumentForApplication()` |
| BR-068 | Rút được ở mọi trạng thái trước quyết định (kể cả DRAFT). Link xác nhận chưa trả lời bị đặt hết hạn ngay; các owner đã CONFIRMED (trừ người nộp) được báo qua email `ORG_APPLICATION_WITHDRAWN_NOTICE` | `POST /:id/withdraw` | 409 ORGANIZATION_APPLICATION_ALREADY_DECIDED | `withdrawApplication()` |
| BR-069 | Danh sách owner khi nộp: 1..5 người; email không trùng; người nộp phải có tên; đúng 1 người `isLegalRep`. Lưu nháp chỉ kiểm tra email hợp lệ, họ tên 1..200, ≤ 5 dòng, không trùng | `PUT /:id`, `POST /:id/submit` | 422 AT_LEAST_ONE_OWNER / TOO_MANY_OWNERS / DUPLICATE_OWNER_EMAIL / SUBMITTER_MUST_BE_OWNER / EXACTLY_ONE_LEGAL_REP | `owner-candidates.ts > validateOwnerList(), normalizeOwnerInputs()` |

## 4. Thẩm định đơn và Blue Tick

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-070 | Claim bị chặn nếu đơn đã có reviewerId khác người claim. Claim chỉ ghi reviewer, **không đổi status** | `PUT /admin/organization-applications/:id/claim` | 409 ORGANIZATION_APPLICATION_CLAIMED | `organization-application-admin.service.ts > claim()` |
| BR-071 | Admin không bao giờ thấy đơn DRAFT hoặc AWAITING_OWNER_CONFIRMATION (danh sách loại ra, xem chi tiết trả 404). Claim, request-info, decision chỉ khi PENDING_REVIEW; đơn đã APPROVED, REJECTED, WITHDRAWN trả ALREADY_DECIDED | claim, request-info, decision, list, get | 404; 409 NOT_PENDING_REVIEW / ORGANIZATION_APPLICATION_ALREADY_DECIDED | `loadPendingReview()`, `organization-application.repository.ts > HIDDEN_FROM_ADMIN_STATUSES` |
| BR-072 | Yêu cầu bổ sung cần message không rỗng, ≤ 5000 ký tự | request-info | 400 | `requestMoreInfo()` |
| BR-073 | decision ∈ APPROVE, REJECT; REJECT bắt buộc có reject_reason | decision | 400 | `decide()`, `reject()` |
| BR-074 | APPROVE bắt buộc chọn lane A hoặc B | decision | 400 "lane must be A or B when approving" | `approve()` |
| BR-075 | Nếu không miễn giấy tờ thì đơn phải có ≥ 1 giấy tờ; nếu miễn thì phải ghi lý do | decision | 400 | `approve()` |
| BR-076 | Profile phải có name và logo mới được duyệt | decision | 400 | `approve()` |
| BR-077 | Blue Tick (`trustTier=VERIFIED`) = `grant_blue_tick`, mặc định bằng `lane === A`. Lane B đặt `verificationExpiresAt` = +365 ngày; `domainVerified = laneA && documentsWaived` | decision APPROVE | — | `approve()`, `DC/organization-trust.ts > LANE_B_VERIFICATION_VALID_DAYS` |
| BR-078 | Mỗi lần mở giấy tờ đều ghi audit DOCUMENT_VIEWED — cả admin lẫn người nộp (người nộp mở bằng tracking token, event không có actor); link tải có TTL 5 phút; giấy tờ đã bị purge thì không mở được | `GET /admin/…/:id/documents/:docId/file`, `GET /organization-applications/:id/documents/:docId/file?token=` | 404 | `openDocument()`, `openDocumentForApplicant()` |
| BR-079 | Onboarding sau duyệt (outbox `ORG_OWNER_ONBOARD`, một event mỗi owner, dedupKey theo candidate): user còn `PENDING_ACTIVATION` → token kích hoạt mới 72h (thu hồi token cũ) + email `ACCOUNT_ACTIVATION`; user đã active → email `ORG_OWNER_ATTACHED`, **không bao giờ** gửi link đặt lại mật khẩu | outbox relay | throw → retry | `organization-owner-onboard.publisher.ts` |

## 4b. Owner xác nhận và membership tổ chức

> Dải mã 300–329. Thiết kế: [ORG_OWNERSHIP_FLOW.md](ORG_OWNERSHIP_FLOW.md). Hằng số: `DC/organization-trust.ts` (`OWNER_ORG_LIMIT = 3`, `MAX_OWNERS_PER_APPLICATION = 5`, `OWNER_CONFIRM_TTL_DAYS = 14`, `OWNER_CONFIRM_RESEND_COOLDOWN_MS = 1h`, `MAX_PENDING_INVITES_PER_EMAIL = 2`).

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-300 | Kiểm tra chặn sớm khi nộp, **trước khi gửi email nào**: owner có tài khoản status 2 → OWNER_SUSPENDED; owner đã có ≥ 3 membership vai owner (tổ chức chưa xoá) → OWNER_QUOTA_EXCEEDED; không gọi được identity → 503 | `POST /:id/submit` | 422 (message kèm email) / 503 | `organization-application.service.ts > assertOwnersEligible()` |
| BR-301 | Chống spam: một email không được đang có tên (chưa bị gỡ) ở ≥ 2 đơn **khác** đang AWAITING_OWNER_CONFIRMATION / PENDING_REVIEW / NEEDS_REVISION | `POST /:id/submit` | 422 TOO_MANY_PENDING_INVITES | `organization-application.repository.ts > countOtherCandidacies()` |
| BR-302 | Email đã chọn "chặn mọi lời mời sau này" (`owner_invite_blocks`) không được ghi làm owner, trừ chính người nộp | `POST /:id/submit` | 422 OWNER_INVITE_BLOCKED | `findBlockedEmails()` |
| BR-303 | Owner đã DECLINED phải được gỡ hoặc thay trước khi nộp lại | `POST /:id/submit` | 422 OWNER_DECLINED_MUST_BE_REPLACED | `submitApplication()` |
| BR-304 | Người nộp được tự đánh dấu CONFIRMED lúc nộp (hộp thư đã qua OTP), ghi IP và user agent của request nộp. Không còn owner nào chưa xác nhận thì đơn vào thẳng PENDING_REVIEW | `POST /:id/submit` | — | `submitApplication()` |
| BR-305 | Link xác nhận: 32 byte ngẫu nhiên, chỉ lưu sha256, TTL 14 ngày; chỉ gửi cho owner chưa có link còn hạn, sau khi transaction commit | submit, resend | — | `owner-candidates.ts > newConfirmToken(), sendConfirmationEmails()` |
| BR-306 | Quy tắc reset: so snapshot (tên, loại tổ chức, email người đại diện pháp lý, tập email owner) với lần nộp trước, không phụ thuộc thứ tự key. Khác → mọi owner CONFIRMED về PENDING kèm link mới, ghi `OWNER_CONFIRMATIONS_RESET`. Sửa mô tả, logo, giấy tờ, kênh thì giữ xác nhận | `POST /:id/submit` | — | `buildConfirmationSnapshot()`, `snapshotsDiffer()` |
| BR-307 | Gửi lại lời mời: đơn AWAITING hoặc NEEDS_REVISION, owner đang PENDING; **không giới hạn số lần**, nhưng cách lần trước ≥ 1 giờ (chốt chống spam duy nhất vì email đi tới người khác); token mới vô hiệu token cũ; hạn đặt lại 14 ngày | `POST /:id/owners/:candidateId/resend` | 429 RESEND_TOO_SOON; 409 APPLICATION_NOT_ACTIVE; 404 | `resendOwnerInvite()` |
| BR-308 | Xác nhận: idempotent (đã CONFIRMED → `already_done`); owner bị gỡ → 404; đã DECLINED → 409; đơn không ở AWAITING / NEEDS_REVISION → 409 (kiểm tra trước hạn); hết hạn → 410; ghi IP, user agent. Lần xác nhận cuối khi đơn đang AWAITING chuyển PENDING_REVIEW **trong cùng transaction**, có row lock trên đơn | `POST /owner-confirmations/:token/confirm` | 404 OWNER_CONFIRMATION_NOT_FOUND; 409 ALREADY_DECLINED / APPLICATION_NOT_ACTIVE; 410 CONFIRM_EXPIRED | `owner-confirmation.service.ts > confirm()` |
| BR-309 | "Tôi không liên quan": owner đã CONFIRMED → 409; bấm lần hai không làm gì; đơn về NEEDS_REVISION với `reviewNote`; `block_future` ghi `owner_invite_blocks`; lý do ≤ 1000 ký tự; người nộp nhận email `ORG_OWNER_DECLINED` | `POST /owner-confirmations/:token/decline` | 409 ALREADY_CONFIRMED / APPLICATION_NOT_ACTIVE | `decline()` |
| BR-310 | Sweeper mỗi giờ (`OWNER_CONFIRMATION_EXPIRY_INTERVAL_MS`, tắt bằng `OWNER_CONFIRMATION_EXPIRY_ENABLED=false`): owner PENDING quá hạn trên đơn AWAITING → EXPIRED, đơn về NEEDS_REVISION, người nộp nhận email `ORG_OWNER_CONFIRMATION_EXPIRED` | worker incident | — | `owner-confirmation-expiry.job.ts`, `expireOverdue()` |
| BR-311 | Lưu nháp đồng bộ danh sách owner theo email: dòng biến mất được đánh `removedAt` (**không xoá**), ghi `OWNER_CANDIDATE_REMOVED`; thêm lại thì về PENDING, xoá phản hồi cũ | `PUT /:id` | — | `syncOwners()` |
| BR-312 | Duyệt: identity ensure-users **trước** transaction; có owner status 2 → OWNER_SUSPENDED. Trong transaction: khoá đơn, kiểm lại PENDING_REVIEW và mọi owner CONFIRMED; với từng owner khoá advisory theo userId rồi đếm lại trần 3 tổ chức (vượt → rollback toàn bộ); tạo membership `LEGAL_REPRESENTATIVE` / `OWNER`, `source = APPLICATION_APPROVAL` | decision APPROVE | 409 NOT_PENDING_REVIEW / OWNERS_NOT_ALL_CONFIRMED; 422 OWNER_QUOTA_EXCEEDED / OWNER_SUSPENDED; 503 | `organization-application-admin.service.ts > approve()`, `organization-membership.service.ts > assertOwnerQuota(), grantMembership()` |
| BR-313 | Tổ chức mới chỉ có `isEmailVerified = true` khi contactEmail trùng `submitterEmail` đã qua OTP | decision APPROVE | — | `approve()` |
| BR-314 | Không gán vai vào tổ chức không ACTIVE hoặc đã xoá | mọi grant | 409 | `grantMembership()` |
| BR-315 | Bất biến DB: mọi tổ chức chưa xoá luôn có ≥ 1 membership chưa xoá vai `LEGAL_REPRESENTATIVE` / `OWNER`; kiểm tra lúc COMMIT bằng constraint trigger deferred | insert/update tổ chức, update/delete membership | Postgres raise `ORG_MUST_HAVE_OWNER` | migration `20260926100000_org_multi_owner`, `isOrgMustHaveOwnerViolation()` |
| BR-316 | Một người chỉ có một vai trong một tổ chức (khoá chính `(organization_id, user_id)`) | membership | — | `schema.prisma > OrganizationMember` |
| BR-317 | OTP mở đơn DRAFT **mới** thì gửi email `ORG_APPLICATION_DRAFT_STARTED` kèm link trình soạn nháp (tracking token 180 ngày); mở lại đơn đang có thì không gửi. Gửi lỗi không chặn việc mở nháp | `POST /email-otp/verify` | — | `organization-application.service.ts > openDraftForEmail()` |
| BR-318 | Bấm **"Lưu nháp"** (client gửi `notify_submitter: true`; "Continue"/"Nộp" thì không) → gửi `ORG_APPLICATION_DRAFT_UPDATED` tới email người nộp kèm link trình soạn nháp; tối đa **1 email / giờ / hồ sơ** (đếm theo event `DRAFT_UPDATE_NOTIFIED`). Quá hạn mức thì vẫn lưu, `notified=false`. Gửi lỗi không làm hỏng việc lưu | `PUT /:id` | — | `organization-application.service.ts > saveDraft() → notifyDraftUpdated()` |

## 5. Tổ chức

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-080 | Tạo tổ chức trực tiếp chỉ qua API key nội bộ: name ≤ 200, logoUrl bắt buộc, contactEmail bắt buộc, ownerId là UUID; tạo luôn membership OWNER trong cùng transaction | `POST /api/v1/organizations` | 401 / 400 | `INC/modules/organization/organization.controller.ts > createOrganization`, `organization.repository.ts > create()` |
| BR-081 | Tên (không phân biệt hoa thường) + contactEmail là duy nhất trong các tổ chức chưa xoá và chưa bị ban. Rule này **không áp dụng khi duyệt đơn** | tạo, cập nhật tổ chức | 409 ORGANIZATION_ALREADY_EXISTS | `organization.service.ts > assertUniqueNameAndContactEmail()` |
| BR-082 | Slug = tên bỏ dấu, lowercase, gạch nối; trùng thì thêm -2, -3…; không đổi khi đổi tên | tạo tổ chức | 409 "Unable to allocate a unique slug" | `DC/organization-slug.ts`, `allocateUniqueSlug()` |
| BR-083 | Chỉ người có vai owner (`LEGAL_REPRESENTATIVE` hoặc `OWNER`) được sửa tổ chức; phải có ít nhất 1 field; đổi contactEmail thì mất trạng thái xác minh email và hệ thống gửi link mới | `PUT /organizations/:id` | 403; 400 | `updateOrganization()`, `assertOwner()` |
| BR-084 | Admin duyệt tổ chức (status 1) chỉ từ DRAFT, INACTIVE, INREVIEW, PENDING; ban (status 2) chỉ từ DRAFT, PENDING, INREVIEW, ACTIVE và bắt buộc lý do ≤ 5000 | `PUT /organizations/:id/verify` | 400 | `adminVerifyOrganization()` |
| BR-085 | Xin gia nhập: người đã có membership (kể cả owner) không được xin; không được có 2 yêu cầu PENDING | `POST /organizations/:id/join-requests` | 409 / 400 | `createJoinRequest()` |
| BR-086 | Mọi owner xem và xử lý được yêu cầu gia nhập (và đều nhận thông báo có yêu cầu mới); yêu cầu phải đang PENDING; duyệt thì tạo membership role `MEMBER` | `GET /:id/join-requests`, `PUT /join-requests/process` | 403; 409 JOIN_REQUEST_ALREADY_PROCESSED | `processJoinRequest()`, `listJoinRequestsForOwner()` |
| BR-087 | Chỉ người xin được huỷ, và chỉ khi yêu cầu PENDING | `DELETE /join-requests/cancel` | 403 / 400 | `cancelJoinRequest()` |
| BR-088 | Owner không được rời tổ chức (luồng rời / chuyển giao chưa có); owner cuối cùng trả ORG_MUST_HAVE_OWNER; phải là thành viên thì mới rời được | `DELETE /:id/members/me` | 409 ORG_MUST_HAVE_OWNER / 400 | `leaveOrganization()` |
| BR-089 | Tổ chức bị ban (INACTIVE) trả 404 khi xem theo slug | `GET /organizations/by-slug/:slug` | 404 | `getBySlug()` |
| BR-090 | Gửi lại email xác minh: chỉ owner; tổ chức phải có email và email chưa được xác minh | `POST /:id/resend-contact-email` | 403 / 400 / 502 | `resendOrganizationContactVerificationEmail()` |
| BR-091 | Xác minh email liên hệ: email trong token phải trùng contactEmail hiện tại | `GET /organizations/verify-contact-email` | 302 `?error=mismatch` | `confirmOrganizationContactEmail()` |

## 6. Báo cáo sự cố (report)

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-100 | Tạo report: title bắt buộc; latitude và longitude bắt buộc và trong khoảng hợp lệ; severityLevel là số nguyên 1..5; imageUrls ≥ 1 chuỗi không rỗng (không kiểm tra định dạng URL) | `POST /api/v1/reports` | 400 VALIDATION_ERROR | `INC/modules/report/report.controller.ts > createReport` |
| BR-101 | Report mới có status PENDING (12), isVerify=false, aiVerified=false; hệ thống enqueue phân tích AI và dịch | tạo report | — | `report.service.ts > createReport()` |
| BR-102 | Tìm kiếm: statuses ≤ 25 giá trị; maxDistance mặc định 50000 m; limit 1..100 (mặc định 10); có đủ lat và lng thì lọc theo khoảng cách (PostGIS) và mặc định sắp theo distance | `GET /reports/search`, `/my` | 400 | `report.repository.ts > searchWithDistance()` |
| BR-103 | Report bị ban (INACTIVE) trả 404 khi xem chi tiết | `GET /reports/:id` | 404 REPORT_NOT_FOUND | `getReportDetail()` |
| BR-104 | Chỉ chủ report được sửa, thêm ảnh, xoá ảnh hoặc xoá report; admin bị cấm; report đã bị ban không sửa được | `PUT /:id`, `POST /:id/media`, `DELETE /:id/media/:mid`, `DELETE /:id` | 403 | `assertReporterMayEditReport()` |
| BR-105 | Thêm ảnh sẽ đặt lại status=PENDING và aiVerified=false, rồi phân tích lại | `POST /:id/media` | — | `addReportImages()` |
| BR-106 | Admin duyệt report: status → TODO (21), isVerify=true; đã duyệt thì no-op | `PUT /:id/verify` | — | `adminVerifyReport()` |
| BR-107 | Admin ban report bắt buộc có lý do ≤ 5000 | `PUT /:id/ban` | 400 | `adminBanReport()` |
| BR-108 | Đánh dấu hoàn thành: đã COMPLETED thì no-op; điểm xanh = env `REPORT_COMPLETION_GREEN_POINTS` (mặc định 0); chỉ phát outbox khi report có userId | `PUT /:id/mark-done` | — | `adminMarkReportDone()` |
| BR-109 | Media file theo ids: chỉ trả khi người xem là chủ report hoặc report đã được duyệt | `GET /reports/media-files/by-ids` | File không đủ quyền bị bỏ qua (không báo lỗi) | `report_media.repository.ts > findManyByIdsVisibleToViewer()` |
| BR-110 | Chỉ report ở TODO (21) và chưa thuộc campaign nào mới gắn được vào campaign; khi gắn thì chuyển INPROCESS (22); gỡ ra thì trở về TODO | tạo / sửa / ban / xoá campaign | 400 "One or more reportIds are invalid" | `INC/modules/campaign/campaign.service.ts > validateReportIds(), assignReportsToCampaign()` |
| BR-111 | Title và description hiển thị ưu tiên bản tiếng Việt (`titleVi ?? title`) | response report | — | `report.entity.ts > toReportResponse()` |
| BR-112 | Phân tích AI cần ≥ 1 ảnh hợp lệ; predict không trả kết quả → job lỗi; gợi ý (recommendation) lỗi thì chỉ cảnh báo; timeout 45s | worker ANALYZE_REPORT | retry | `report-ai-analysis.service.ts > analyzeReport()` |

## 7. Vote và lưu

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-130 | Vote kiểu toggle: upvote (đang 1 → 0, khác → 1); downvote (đang -1 → 0, khác → -1); mỗi user có 1 vote cho mỗi tài nguyên | `POST /incident/votes/upvote|downvote` | — | `INC/modules/vote/vote.service.ts` |
| BR-131 | resourceType ∈ report, campaign; tài nguyên phải tồn tại (không xét status) | vote, save | 400 / 404 | `ensureVotableResource()`, `ensureSaveableResource()` |
| BR-132 | Chỉ phát event mốc vote khi: loại là report, giá trị mới = 1, report có chủ; dedup theo (reportId, voteCount) | upvote | — | `emitReportVoteMilestoneIfNeeded()` |
| BR-133 | Lưu kiểu toggle: chưa có → tạo; đã xoá mềm → khôi phục; đang lưu → xoá mềm | `POST /incident/saved-resources/save` | — | `saved_resource.service.ts > save()` |

## 8. Chiến dịch

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-150 | Tạo campaign: organizationId là UUID; title bắt buộc; banner ≤ 2048; ngày theo ISO8601; detailAddress ≤ 255; toạ độ hợp lệ; radiusKm ≥ 0; difficulty là số nguyên ≥ 1 | `POST /api/v1/campaigns` | 400 | `campaign.controller.ts > createCampaign` |
| BR-151 | Chỉ người có vai owner (`LEGAL_REPRESENTATIVE` / `OWNER`) của tổ chức được tạo campaign; tổ chức phải tồn tại (không xét status) | tạo campaign | 404 / 403 "Only an organization owner can create campaigns" | `campaign.service.ts > createCampaign()`, `organization_member.repository.ts > isOwner()` |
| BR-152 | difficulty phải có tier tương ứng ở reward-service | tạo, sửa campaign | 400 "Invalid campaign difficulty…" | `createCampaign()`, `updateCampaign()` |
| BR-153 | Campaign mới có status PENDING (12); người tạo tự động là manager | tạo campaign | — | `createCampaign()` |
| BR-154 | Chỉ `createdBy` được sửa hoặc xoá campaign; sửa có thể đặt status bất kỳ; xoá thì gỡ report | `PUT /:id`, `DELETE /:id` | 403 "Only campaign manager can modify campaign" | `updateCampaign()`, `deleteCampaign()`, `ensureOwner()` |
| BR-160 | Admin duyệt (→ 1) chỉ từ {12, 4, 5, 2}; ban (→ 2) chỉ từ {12, 4, 5, 1}, bắt buộc lý do ≤ 5000; đang ACTIVE mà duyệt lại thì no-op | `PUT /campaigns/:id/verify` | 400 | `adminVerifyCampaign()` |
| BR-161 | Duyệt campaign có toạ độ thì mời người dân trong bán kính **5 km** (trừ admin, người tạo, manager) | verify | — | `notifyNearbyCitizensToJoinApprovedCampaign()` |
| BR-162 | Ban campaign sẽ gỡ các report (INPROCESS → TODO) | verify status 2 | — | `banCampaignAndUnlinkReports()` |
| BR-165 | Gửi hoàn thành: người gửi phải có trong bảng manager; campaign ở ACTIVE hoặc INREVIEW; **mọi task đã COMPLETED** | `PUT /:id/mark-done` | 400 "Some tasks is not completed"; 500 khi status sai | `submitCampaignCompletionForAdminApproval()` |
| BR-166 | Duyệt hoàn thành chỉ khi campaign ở WAITING_CONFIRMED (7); từ chối bắt buộc lý do | `PUT /:id/completion-review` | 400 | `adminFinalizeCampaignCompletion()`, `adminRejectCampaign()` |
| BR-167 | Điểm thưởng hoàn thành = `tier.greenPoints` cho mỗi volunteer **vừa được APPROVED vừa đã check-in**; không có ai đủ điều kiện thì không phát event | completion-review approve | — | `adminFinalizeCampaignCompletion()` |
| BR-168 | Duyệt hoàn thành sẽ chuyển report và SOS của campaign sang COMPLETED; từ chối thì campaign trở về ACTIVE | completion-review | — | như trên |
| BR-170 | Xin tham gia: campaign phải tồn tại (không xét status); người xin chưa có yêu cầu nào chưa bị xoá | `POST /campaigns/volunteers/join-requests` | 404; 409 "Join request already exists" | `campaign_joining_request.service.ts > createJoinRequest()` |
| BR-171 | Chỉ manager (có trong bảng) được xem và xử lý yêu cầu; yêu cầu phải PENDING | `GET/PUT /volunteers/join-requests(/process)` | 403; 409 | `processJoinRequest()` |
| BR-172 | Duyệt tham gia phải còn chỗ: số APPROVED < `maxVolunteers` của tier (null nghĩa là không giới hạn) | process approve | 400 "Campaign volunteer capacity exceeded…" | `INC/modules/reward/reward-service.client.ts > assertCampaignHasCapacityForJoinApproval()` |
| BR-173 | Từ chối tham gia = xoá mềm yêu cầu, nên người đó có thể xin lại ngay | process reject | — | `processJoinRequest()` |
| BR-174 | Huỷ yêu cầu tham gia: chỉ chính chủ, khi PENDING | `DELETE /volunteers/join-requests/cancel` | 403 / 409 | `cancelJoinRequest()` |
| BR-175 | Thêm hoặc gỡ manager, CRUD task, giao task, tạo QR: `createdBy` hoặc manager (`canManageCampaign`) | các endpoint manager/task/QR | 403 | `campaign_manager.service.ts > canManageCampaign()` |
| BR-176 | Task: title bắt buộc; priority 1..3 (mặc định 2); task mới ở TODO; lần đầu giao thì chuyển INPROCESS | tạo, giao task | 400 | `campaign_task.service.ts` |
| BR-177 | Chỉ giao task cho volunteer đã APPROVED, và không giao trùng | `POST /tasks/:id/assign` | 403 / 409 | `assignTask()` |
| BR-178 | Kết quả task: volunteer được giao hoặc manager được cập nhật; file bị thay toàn bộ | `PUT /tasks/:id` (result) | 403 | `updateTaskResult()` |
| BR-179 | Volunteer chỉ đổi status của task được giao cho mình | `PUT /tasks/:id/status` | 403 "You are not assigned to this task" | `updateTaskStatusByVolunteer()` |
| BR-180 | QR điểm danh chỉ tạo được khi campaign ACTIVE; JWT TTL 1 giờ | `POST /:id/attendance-qr` | 400 / 403 | `campaign_attendance.service.ts > issueAttendanceQr()` |
| BR-181 | Check-in: token đúng campaign, campaign ACTIVE, người quét đã APPROVED; mỗi người check-in 1 lần | `POST /:id/attendance-check-in` | 400 / 403 | `checkInWithQrToken()` |
| BR-182 | Xác nhận sạch: value ∈ {1, -1}; campaign ở 7 hoặc 17; gửi lại cùng giá trị thì huỷ (thành 0) | `POST /:id/completion-verification` | 400 | `campaign_completion_verification.service.ts > submit()` |
| BR-183 | Submission: chỉ manager được tạo và duyệt; chỉ người nộp được thêm kết quả; chỉ duyệt khi đang ở 9, 6 hoặc 12 | submission endpoints | 403 / 409 | `campaign_submission.service.ts` |
| BR-184 | Danh sách phân trang: page ≥ 1, limit 1..100; by-ids tối đa 100 UUID | các GET campaign | 400 | `campaign.controller.ts` |

## 9. SOS

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-190 | SOS: campaignId là UUID; content không rỗng, ≤ 2000; phone khớp `^\+?\d{7,15}$`; campaign tồn tại, **ACTIVE** và có toạ độ | `POST /api/v1/sos` | 400 / 404 | `INC/modules/sos/sos.controller.ts`, `sos.service.ts > create()` |
| BR-191 | SOS lấy toạ độ và địa chỉ của campaign, status=1 | tạo SOS | — | `create()` |
| BR-192 | Giải quyết SOS: id là số nguyên ≥ 1; đã COMPLETED thì no-op; không kiểm tra quyền | `PUT /api/v1/sos/:id/solved` | 400 / 404 | `solveSos()` |

## 10. Thông báo

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-200 | Enqueue: type là chuỗi không rỗng; kind thuộc `NotificationKind`; website bắt buộc userId; email với 6 kind đơn tổ chức bắt buộc `payload.toEmail`, các kind khác bắt buộc userId | `POST /api/v1/notifications/jobs` | 400 | `NS/modules/notification/notification.controller.ts > enqueue()`, `direct-email-kinds.ts` |
| BR-201 | Thông báo website gửi cho user sẽ lọc theo preference của người nhận. Kind admin-only (đơn tổ chức, duyệt/từ chối, reset password…) luôn được gửi | incident trước khi enqueue | Người tắt thông báo không nhận | `DC/notification-preferences.ts > isNotificationEnabledForUser()` |
| BR-202 | Email: `payload.locale == "vi"` thì dùng template tiếng Việt, ngược lại dùng tiếng Anh; website render cả hai ngôn ngữ | worker | — | `NS/channels/email/email.channel.ts`, template engine |
| BR-203 | `SMTP_HOST` rỗng thì không gửi email, vẫn ghi DB với `skippedReason = smtp_not_configured` | worker email | — | `email.provider.ts > send()` |
| BR-204 | Retry tối đa 5 lần, backoff `min(900, 30·2^(n-1))` giây | worker | FAILED | `ecolink-server/shared/da2-queue/src/core/queue-worker.ts` |
| BR-205 | Danh sách thông báo: chỉ loại WEBSITE của chính user; limit 1..200 (mặc định 50) | `GET /api/v1/notifications/my` | 400 | `notification.service.ts > listForUser()` |
| BR-206 | Đánh dấu đã đọc: thông báo phải là WEBSITE và thuộc user | `PATCH /:id/read` | 404 | `markRead()` |

## 11. Quà tặng

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-220 | Người không phải admin chỉ thấy quà đang active; lọc theo khoảng điểm yêu cầu greenPointsMin ≤ greenPointsMax | `GET /api/v1/gifts` | 400 | `RW/modules/gift/gift.service.ts > listGifts()`, `gift.api.routes.ts` |
| BR-221 | Tạo quà: name 1..255, imageUrl là URL, description bắt buộc, greenPoints ≥ 0, stockRemaining null (vô hạn) hoặc ≥ 0 | `POST /api/v1/gifts` (admin) | 400 | `gift.api.routes.ts` |
| BR-222 | Đổi quà: phone 7..32 ký tự theo regex `^[0-9+\-()\s.]+$`; pickupLocation 1..1000 | `POST /gifts/:id/redeem|exchange` | 400 | `gift.api.routes.ts` |
| BR-223 | Quà phải tồn tại và active; còn hàng (nếu có giới hạn tồn kho, trừ nguyên tử) | redeem | 404; 422 "Gift is out of stock" | `gift.service.ts > redeem()` |
| BR-224 | Giá thực tế = `ceil(greenPoints × (10000 − discountBps)/10000)`, discountBps = mức giảm tốt nhất từ badge của season hiện tại | redeem | — | `redeem()`, `RW/modules/gamification/badge.service.ts > getBestStoreDiscountBps()` |
| BR-225 | Phải đủ **SP còn hạn**; trừ theo FIFO của ngày hết hạn | redeem | 422 "Insufficient spendable points (SP)" | `RW/modules/gamification/sp-wallet.util.ts` |
| BR-230 | Trạng thái đơn đổi quà: PROCESSING → SHIPPED/CANCELLED; SHIPPED → DELIVERED/CANCELLED; DELIVERED và CANCELLED là trạng thái cuối; chuyển sang cùng trạng thái thì no-op | `PATCH /admin/gift-redemptions/:id/status` | 422 "Cannot change redemption status from X to Y" | `gift.service.ts > updateRedemptionStatus()` |
| BR-231 | Huỷ đơn đã tiêu điểm thì hoàn SP (1 lần, lô mới với hạn mới); không hoàn tồn kho | như trên | — | `updateRedemptionStatus()` |
| BR-232 | Sửa difficulty: tên 1..64 ký tự; maxVolunteers null hoặc ≥ 1; greenPoints ≥ 0 | `PUT /api/v1/difficulties/:id` (admin) | 400 / 404 | `RW/modules/difficulty/difficulty.service.ts > updateById()` |

## 12. Điểm và gamification

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-240 | Cộng điểm xanh idempotent theo (user, type, resourceId, resourceType); điểm ≤ 0 thì bỏ qua | worker reward-intake | Trùng thì bỏ qua | `RW/modules/green-point/green-point-ledger.util.ts > applyGreenPointLedgerCredit()` |
| BR-241 | Mỗi lần cộng điểm xanh sẽ cộng SP bằng đúng số đó; SP hết hạn sau `expirationDays` (mặc định 90 ngày) | worker | — | `RW/modules/gamification/sp-credit.util.ts` |
| BR-242 | Điểm từ campaign → VRP; từ report hoặc mốc vote → CRP; RP phải gắn với season hiện tại, không có season thì không ghi RP | worker | — | `rp-credit.util.ts > applyRankingPointCredit()` |
| BR-243 | Season hiện tại = season ACTIVE bao trùm thời điểm hiện tại → nếu không có thì season ACTIVE mới nhất → nếu không có thì season mới nhất | nhiều nơi | — | `season.service.ts > getCurrentSeason()` |
| BR-244 | Điểm mốc vote: với mỗi ngưỡng ≤ voteCount, điểm = baseReportPoint × (vị trí ngưỡng + 1) | worker | — | `gamification-config.service.ts > resolveReportVoteMilestoneCredits()` |
| BR-245 | Payload của các loại event phải đúng cấu trúc (campaignId, credits[]…) | worker | throw → retry | `RW/modules/green-point/strategies/*` |
| BR-246 | Bảng xếp hạng: metric ∈ crp, vrp, org_aggregate; chỉ tính điểm > 0; season INACTIVE thì đọc snapshot | `GET /gamification/leaderboards/:metric` | 400 | `gamification-leaderboard.service.ts` |
| BR-247 | Tạo season: kind ∈ MONTHLY, QUARTERLY; ngày parse được; status 1/2 (mặc định ACTIVE). Không kiểm tra startsAt < endsAt, không kiểm tra trùng | `POST /admin/seasons` | 400 | `gamification.controller.ts > adminCreateSeason()` |
| BR-248 | Finalize: season phải có status 1 hoặc 2; `openNext` cần startsAt < endsAt và không có season ACTIVE khác | `POST /admin/seasons/:id/finalize` | 400 / 404 / 422 | `season.service.ts > finalizeSeason()` |
| BR-249 | Finalize season ACTIVE: chụp snapshot bảng xếp hạng, chi SP theo payout tier (CRP, VRP), rồi chuyển INACTIVE | finalize | — | `freezeSeasonTx()`, `applyPayoutTiersTx()` |
| BR-250 | Payout tier: metric ∈ CRP/VRP/ORG_AGGREGATE; rankMin ≥ 1; rankMax ≥ rankMin; spAmount ≥ 0 | `POST /admin/gamification/payout-tiers` | 400 | `adminCreatePayoutTier()` |
| BR-251 | Badge: name và category bắt buộc; `rulesConfig` là AST với agg COUNT/SUM, toán tử gt/gte/lt/lte/eq/neq, bảng và cột phải có trong metric metadata; `reward` chỉ nhận key discountBps (0..10000), bonus_sp, partner_tier_codes, perks | `POST/PATCH /admin/gamification/badges` | 400 | `RW/modules/metrics/metric-rules.validation.ts`, `badge-definition.validation.ts` |
| BR-252 | Slug của badge là duy nhất, tự sinh từ tên, bị khoá khi publish; badge đã được cấp thì không đổi category hoặc scope | badge | 400 | `badge.service.ts > allocateUniqueSlug(), patchDefinition()` |
| BR-253 | Cấu hình điểm: baseReportPoint ≥ 0; ngưỡng là mảng số nguyên; expirationDays ≥ 1 | `PATCH point-rules`, `sp-rules` | 400 | `adminPatchPointRules()`, `adminPatchSpRules()` |
| BR-254 | Ước tính thưởng campaign: difficultyLevel là số nguyên ≥ 1 và phải tồn tại | `GET /gamification/campaign-reward-estimate` | 400 / 404 | `getCampaignRewardEstimate()` |

## 13. AI và dịch

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-270 | agentId ∈ ecolink_assistant, translation_assistant | `POST /api/v1/chat/conversations` | 400 | `AI/chat/router.py > create_conversation()` |
| BR-271 | Chỉ chủ hội thoại mới đọc hoặc stream được | messages, stream | 404 / event error | `AI/repositories/chat.py > get_conversation_for_user()` |
| BR-272 | Mỗi tin nhắn phải có content hoặc mediaIds (≤ 10, UUID); media phải thuộc chính user | stream | 400 / 422 | `router.py > stream_message()` |
| BR-273 | Tối đa 8 vòng gọi tool trong 1 lượt chat | stream | event error "Too many tool rounds" | `AI/chat/service.py > stream_chat_turn()` |
| BR-274 | Tool tạo report cần ≥ 1 ảnh; severity ∈ {1, 2}; toạ độ mặc định (1, 1) | tool `create_report` | Trả lỗi dạng JSON cho LLM | `AI/tools/report_api.py` |
| BR-275 | Dịch: content không rỗng; kết quả phải là JSON có `detected_language` ∈ {vi, en}; trường ngôn ngữ nguồn bị ép bằng input gốc | `/internal/v1/translate`, `/api/v1/chat/translate` | 400 / 500 | `AI/chat/service.py > translate_text()` |
| BR-276 | Dịch nền: ai-service lỗi thì ghi text gốc vào cả 2 cột và job vẫn thành công | TranslationWorker (incident, reward) | — | `INC/queue/worker/translation-worker.ts`, `RW/queue/workers/translation.worker.ts` |
| BR-277 | Caption Facebook: campaignTitle 1..500, volunteers ≤ 80; không được đưa email vào bài; tối đa khoảng 900 ký tự | `POST /api/v1/social/campaign-facebook-caption` | 422 / 500 | `AI/social/*` |

## 14. Kiểm tra phía client (chỉ có ở UI)

| ID | Rule | Áp dụng ở đâu | Hệ quả khi vi phạm | File |
|---|---|---|---|---|
| BR-290 | Đăng nhập và đăng ký: password ≥ 6 (server yêu cầu ≥ 8 khi đăng ký); name ≥ 3; phải tick điều khoản | `/sign-in`, `/sign-up` | Form báo lỗi | `FE/src/pages/*` (xem 06-frontend.md) |
| BR-291 | Report: 1–10 ảnh; ảnh được nén (cạnh ≤ 1280px, JPEG 0.5); bắt buộc chọn vị trí | `/incidents/create` | Form báo lỗi | `FE/libs/compressImage.ts` |
| BR-292 | Campaign: title ≤ 200; difficulty 1–4; chỉ chọn được report TODO; không kiểm tra ngày bắt đầu và kết thúc | `/campaigns/create` | Form báo lỗi | 06-frontend.md |
| BR-293 | Task: giờ kết thúc phải sau giờ bắt đầu; task COMPLETED phải có kết quả; ≤ 20 media; video ≤ 100MB | popover task | Form báo lỗi | `FE/components/client/shared/PopoverCreateUpdateTask.tsx` |
| BR-294 | Chat: tối đa 8 ảnh | AiChatWidget | — | `FE/components/client/ai-chat/*` |
| BR-295 | Đơn tổ chức: OTP đúng 6 số; logo bắt buộc; kênh đầu tiên bắt buộc; email liên hệ hợp lệ; danh sách owner kiểm tra giống BR-069; số điện thoại người đại diện bắt buộc, số giấy tờ bắt buộc nếu chưa lưu; không bắt buộc phải có ≥ 1 giấy tờ | `/organizations/apply`, `/organizations/apply/edit/:id` | Form báo lỗi | `FE/app/(pages)/(main)/organizations/apply/_services/application.service.ts > validateOwnerList()` |

---

**Tổng số rule đã ghi nhận: 184** (BR-001…BR-318, có các khoảng trống dành sẵn trong từng dải).
