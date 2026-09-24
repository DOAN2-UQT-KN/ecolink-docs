# 04 — Máy trạng thái

> Mã số trạng thái theo `GlobalStatus` (`ecolink-server/shared/da2-constants/src/global-status.ts`). Xem bảng đầy đủ ở [01-data-model.md §6.1](01-data-model.md).
> "Mgr(table)" nghĩa là người dùng có dòng trong `campaign_managers`. "canManage" nghĩa là `createdBy` hoặc manager (`campaign_manager.service.ts > canManageCampaign()`).
> Đường dẫn viết tắt giống [02-business-flows.md](02-business-flows.md).

## 1. Report (`reports.status`, `isVerify`, `aiVerified`)

```mermaid
stateDiagram-v2
  [*] --> PENDING_12: user tạo report
  PENDING_12 --> TODO_21: admin verify
  PENDING_12 --> INACTIVE_2: admin ban
  TODO_21 --> INACTIVE_2: admin ban
  INACTIVE_2 --> TODO_21: admin verify (gỡ ban)
  TODO_21 --> INPROCESS_22: gắn vào campaign
  INPROCESS_22 --> TODO_21: campaign bị ban / xoá / gỡ report
  INPROCESS_22 --> COMPLETED_17: admin duyệt hoàn thành campaign
  TODO_21 --> COMPLETED_17: admin mark-done
  PENDING_12 --> COMPLETED_17: admin mark-done (không kiểm tra)
  TODO_21 --> PENDING_12: chủ report thêm ảnh
  INPROCESS_22 --> PENDING_12: chủ report thêm ảnh
  COMPLETED_17 --> [*]
```

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → PENDING 12 | User đăng nhập, hoặc qua tool chat AI | BR-100 | Tạo media và report_media_files; enqueue ANALYZE_REPORT và TRANSLATE_TEXT | `INC/modules/report/report.service.ts > createReport()` |
| PENDING / INACTIVE / … → TODO 21 (isVerify=true) | Admin | Chưa được duyệt, hoặc đang bị ban | Thông báo REPORT_APPROVED cho chủ report | `adminVerifyReport()` |
| bất kỳ (≠ 2) → INACTIVE 2 | Admin | Có lý do | Thông báo REPORT_REJECTED. Từ 2 sang 2 thì chỉ đổi lý do | `adminBanReport()` |
| bất kỳ (≠ 17) → COMPLETED 17 | Admin | Không kiểm tra trạng thái nguồn | Outbox REPORT_COMPLETION_GREEN_POINTS; thông báo REPORT_STATUS | `adminMarkReportDone()` |
| TODO 21 → INPROCESS 22 | Owner tổ chức (tạo campaign) hoặc `createdBy` (sửa campaign) | Report chưa thuộc campaign nào | Gán campaignId | `INC/modules/campaign/campaign.service.ts > assignReportsToCampaign()` |
| INPROCESS 22 → TODO 21 | `createdBy` (xoá campaign / đổi reportIds) hoặc admin (ban campaign) | — | campaignId = null | `deleteCampaign()`, `banCampaignAndUnlinkReports()`, `updateCampaign()` |
| INPROCESS 22 → COMPLETED 17 | Admin duyệt hoàn thành campaign | Campaign đang ở 7 | **Không** phát outbox điểm cho report | `adminFinalizeCampaignCompletion()` |
| bất kỳ (không bị ban) → PENDING 12, aiVerified=false | Chủ report | Report không bị ban | Enqueue ANALYZE_REPORT. Nếu `isVerify` đã true thì report bị kẹt (99) | `addReportImages()` |
| aiVerified false → true | Worker | Có kết quả predict | aiRecommendation, ai_analysis_logs | `report-ai-analysis.service.ts > analyzeReport()` |
| → xoá mềm | Chủ report | Report không bị ban | — | `deleteReport()` |

## 2. Campaign (`campaigns.status`)

```mermaid
stateDiagram-v2
  [*] --> PENDING_12: owner tổ chức tạo
  PENDING_12 --> ACTIVE_1: admin duyệt
  PENDING_12 --> INACTIVE_2: admin ban
  ACTIVE_1 --> INACTIVE_2: admin ban
  INACTIVE_2 --> ACTIVE_1: admin duyệt lại
  ACTIVE_1 --> WAITING_CONFIRMED_7: manager gửi hoàn thành (mọi task xong)
  INREVIEW_9 --> WAITING_CONFIRMED_7: manager gửi hoàn thành
  WAITING_CONFIRMED_7 --> COMPLETED_17: admin duyệt hoàn thành
  WAITING_CONFIRMED_7 --> ACTIVE_1: admin từ chối hoàn thành
  COMPLETED_17 --> [*]
  note right of PENDING_12
    createdBy có thể PUT status bất kỳ
    (bỏ qua mọi chuyển trạng thái) - lỗ hổng
  end note
```

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → PENDING 12 | Owner tổ chức | BR-150..BR-153 | Người tạo thành manager; report 21 → 22; TRANSLATE_TEXT; thông báo CAMPAIGN_CREATED cho thành viên | `createCampaign()` |
| 12 / 4 / 5 / 2 → ACTIVE 1 | Admin | — | rejectReason = reason \|\| null; thông báo CAMPAIGN_VERIFY_INVITE cho người dân trong 5 km | `adminVerifyCampaign()` |
| 12 / 4 / 5 / 1 → INACTIVE 2 | Admin | Có lý do | Gỡ report (22 → 21); không gửi thông báo | `banCampaignAndUnlinkReports()` |
| 1 / 9 → WAITING_CONFIRMED 7 | Mgr(table) | Mọi task đều 17 | Thông báo CAMPAIGN_COMPLETION_PENDING_ADMIN (admin trong env) và COMPLETION_VERIFY_INVITE (người dân gần) | `submitCampaignCompletionForAdminApproval()` |
| 7 → COMPLETED 17 | Admin | Mọi task 17; có tier | Report và SOS → 17; outbox CAMPAIGN_COMPLETION_GREEN_POINTS; thông báo CAMPAIGN_DONE và APPROVED_BY_ADMIN | `adminFinalizeCampaignCompletion()` |
| 7 → ACTIVE 1 | Admin | Có lý do | rejectReason; thông báo REJECTED_BY_ADMIN | `adminRejectCampaign()` |
| bất kỳ → bất kỳ | `createdBy` | `PUT /campaigns/:id` với `status` | **Không có kiểm soát** (99) | `updateCampaign()` |
| → xoá mềm | `createdBy` | — | Gỡ report | `deleteCampaign()` |

Không có code nào chuyển campaign sang INREVIEW (9). Trạng thái này chỉ xuất hiện trong điều kiện của mark-done.

## 3. Yêu cầu tham gia campaign (`campaign_joining_requests.status`)

```mermaid
stateDiagram-v2
  [*] --> PENDING_12: user xin tham gia
  PENDING_12 --> APPROVED_14: manager duyệt (còn chỗ)
  PENDING_12 --> Deleted: manager từ chối (xoá mềm)
  PENDING_12 --> Deleted: user huỷ
  APPROVED_14 --> [*]
  Deleted --> [*]
```

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → 12 | User đăng nhập | Chưa có yêu cầu nào chưa bị xoá | Thông báo VOLUNTEER_REQUEST cho các manager | `campaign_joining_request.service.ts > createJoinRequest()` |
| 12 → 14 | Mgr(table) | Còn chỗ theo `maxVolunteers` | VOLUNTEER_APPROVED | `processJoinRequest()` |
| 12 → xoá mềm | Mgr(table), khi từ chối | — | VOLUNTEER_REJECTED | `processJoinRequest()` |
| 12 → xoá mềm | Chính người xin | — | — | `cancelJoinRequest()` |
| 14 → ? | — | Không có API rời hoặc huỷ sau khi được duyệt | — | — |

## 4. Task của campaign (`campaign_tasks.status`)

```mermaid
stateDiagram-v2
  [*] --> TODO_21: manager tạo
  TODO_21 --> INPROCESS_22: giao lần đầu
  INPROCESS_22 --> COMPLETED_17: volunteer/manager cập nhật
  TODO_21 --> COMPLETED_17: cập nhật trực tiếp
  COMPLETED_17 --> INPROCESS_22: cập nhật (không chặn)
```

| Từ → sang | Ai | Điều kiện | File |
|---|---|---|---|
| (mới) → 21 | canManage | — | `campaign_task.service.ts > createTask()` |
| 21 → 22 | Tự động khi assign | Task đang 21 | `assignTask()` |
| bất kỳ → số nguyên bất kỳ | canManage (`PUT /tasks/:id`) hoặc volunteer được giao (`PUT /tasks/:id/status`) | **Không validate giá trị** | `updateTask()`, `updateTaskStatusByVolunteer()` |
| → xoá mềm | canManage | — | `deleteTask()` |

## 5. Submission (`campaign_submissions.status`)

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → INREVIEW 9 | Mgr(table) | — | Gắn kết quả nháp (luôn rỗng) | `campaign_submission.service.ts > createSubmission()` |
| 9 / 6 / 12 → APPROVED 14 hoặc REJECTED 18 | Mgr(table), kể cả chính người nộp | — | Không có | `processSubmission()` |

## 6. SOS (`sos.status`)

```mermaid
stateDiagram-v2
  [*] --> ACTIVE_1: user gửi SOS (campaign ACTIVE)
  ACTIVE_1 --> COMPLETED_17: user bất kỳ bấm solved
  ACTIVE_1 --> COMPLETED_17: admin duyệt hoàn thành campaign
```

| Từ → sang | Ai | Điều kiện | File |
|---|---|---|---|
| (mới) → 1 | User đăng nhập | Campaign ACTIVE, có toạ độ | `INC/modules/sos/sos.service.ts > create()` |
| 1 → 17 | **User đăng nhập bất kỳ** | — | `solveSos()` |
| 1 → 17 | Admin, gián tiếp | Duyệt hoàn thành campaign | `adminFinalizeCampaignCompletion()` |

## 7. Đơn đăng ký tổ chức (`organization_applications.status`)

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: nộp đơn (sau OTP)
  SUBMITTED --> UNDER_REVIEW: admin claim
  SUBMITTED --> NEEDS_MORE_INFO: admin yêu cầu bổ sung
  UNDER_REVIEW --> NEEDS_MORE_INFO: admin yêu cầu bổ sung
  NEEDS_MORE_INFO --> SUBMITTED: người nộp sửa và nộp lại
  NEEDS_MORE_INFO --> UNDER_REVIEW: admin claim
  SUBMITTED --> APPROVED: admin duyệt
  UNDER_REVIEW --> APPROVED: admin duyệt
  NEEDS_MORE_INFO --> APPROVED: admin duyệt
  SUBMITTED --> REJECTED: admin từ chối
  UNDER_REVIEW --> REJECTED: admin từ chối
  NEEDS_MORE_INFO --> REJECTED: admin từ chối
  SUBMITTED --> WITHDRAWN: người nộp rút
  UNDER_REVIEW --> WITHDRAWN: người nộp rút
  NEEDS_MORE_INFO --> WITHDRAWN: người nộp rút
  APPROVED --> [*]
  REJECTED --> [*]
  WITHDRAWN --> [*]
```

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → SUBMITTED | Người nộp ẩn danh (submission token) | BR-055..BR-065 | Event SUBMITTED; burn token; tracking token; email ORG_APPLICATION_RECEIVED | `INC/modules/organization_application/organization-application.service.ts > createApplication()` |
| SUBMITTED / NEEDS_MORE_INFO / UNDER_REVIEW (cùng reviewer) → UNDER_REVIEW | Admin | BR-070 | reviewerId, claimedAt; event CLAIMED | `organization-application-admin.service.ts > claim()` |
| mở → NEEDS_MORE_INFO | Admin (không cần claim) | Có message | reviewNote; event INFO_REQUESTED; tracking token mới; email NEEDS_INFO | `requestMoreInfo()` |
| NEEDS_MORE_INFO → SUBMITTED | Người nộp (tracking token) | Validate lại | Reset reviewer; xoá mềm giấy tờ bị bỏ; event RESUBMITTED (payload `changedFields` — chỉ tên trường đã sửa, không lưu giá trị — cùng `addedDocumentIds`, `removedDocumentIds`) | `updateApplication()` |
| mở → APPROVED | Admin (không cần claim) | BR-074..BR-076 | Tạo organization (trust fields) và channels; outbox ORG_ACCOUNT_PROVISION; event APPROVED (và DOCUMENTS_WAIVED nếu có) | `approve()` |
| mở → REJECTED | Admin | Có lý do | Email ORG_APPLICATION_REJECTED; event REJECTED | `reject()` |
| mở → WITHDRAWN | Người nộp | — | Event WITHDRAWN | `withdrawApplication()` |
| APPROVED (accountProvisionedAt null → có giá trị) | Outbox relay | identity trả user | organization.ownerId, member; event ACCOUNT_PROVISIONED; email ORG_ACCOUNT_ACTIVATION | `organization-account-provision.publisher.ts > publish()` |

`DRAFT` có trong enum nhưng không có đường tạo.

## 8. Tổ chức (`organizations.status`, `trustTier`, `isEmailVerified`, `ownerId`)

```mermaid
stateDiagram-v2
  [*] --> ACTIVE_1: admin duyệt đơn / API nội bộ
  ACTIVE_1 --> INACTIVE_2: admin ban
  INACTIVE_2 --> ACTIVE_1: admin duyệt lại
```

| Trục | Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|---|
| status | (mới) → ACTIVE 1 | Admin (qua duyệt đơn) hoặc service nội bộ | — | Email xác minh (chỉ với luồng nội bộ); TRANSLATE_TEXT (chỉ với luồng nội bộ) | `approve()`, `organization.service.ts > createOrganization()` |
| status | 4 / 2 / 9 / 12 → ACTIVE 1 | Admin | — | Thông báo ORGANIZATION_APPROVED (nếu có owner) | `adminVerifyOrganization()` |
| status | 4 / 12 / 9 / 1 → INACTIVE 2 | Admin | Có lý do | Thông báo ORGANIZATION_REJECTED | `adminVerifyOrganization()` |
| trustTier | (mới) → VERIFIED hoặc NONE | Admin (khi duyệt đơn) | `grant_blue_tick` / lane A | — | `approve()` |
| trustTier | VERIFIED → ? | — | **Không có code gỡ hoặc tạm dừng Blue Tick**, không có job xử lý hết hạn | — | [CHƯA HOÀN THIỆN] |
| kycStatus | NOT_SUBMITTED → APPROVED | Admin (khi duyệt đơn) | — | — | `approve()` |
| isEmailVerified | false → true | Người click link | Token hợp lệ, email khớp | — | `confirmOrganizationContactEmail()` |
| isEmailVerified | true → false | Owner | Đổi contactEmail | Gửi link mới | `updateOrganization()` |
| ownerId | null → userId | Outbox relay | Provision thành công | Upsert member | `publish()` |

## 9. Yêu cầu gia nhập tổ chức (`organization_joining_requests.status`) và thành viên

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → PENDING 12 | User (không phải owner hay thành viên) | BR-085 | VOLUNTEER_REQUEST cho owner | `organization.service.ts > createJoinRequest()` |
| 12 → APPROVED 14 | Owner | — | Upsert organization_members; VOLUNTEER_APPROVED | `processJoinRequest()` |
| 12 → REJECTED 18 | Owner | — | VOLUNTEER_REJECTED | `processJoinRequest()` |
| 12 → xoá mềm | Người xin | — | — | `cancelJoinRequest()` |
| Thành viên: đang hoạt động → xoá mềm | Chính thành viên (không phải owner) | — | — | `leaveOrganization()` |

## 10. Tài khoản người dùng (`users.status`)

```mermaid
stateDiagram-v2
  [*] --> ACTIVE_1: đăng ký / Google
  [*] --> PENDING_ACTIVATION_3: provision tài khoản tổ chức
  PENDING_ACTIVATION_3 --> ACTIVE_1: kích hoạt (đặt mật khẩu)
  ACTIVE_1 --> INACTIVE_2: admin ban
  PENDING_ACTIVATION_3 --> INACTIVE_2: admin ban
  INACTIVE_2 --> ACTIVE_1: kích hoạt bằng token còn hạn (bug)
```

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → 1 | Khách | Email chưa có | — | `ID/modules/auth/auth.service.ts > signup()`, `google.service.ts > handleCallback()` |
| (mới) → 3 | Service nội bộ (incident) | applicationId chưa được provision | Token kích hoạt | `provisionOrgAccount()` |
| 3 → 1 | Người cầm token kích hoạt | Token còn hạn | Đặt password; revoke REFRESH | `activateOrgAccount()` |
| 1 / 3 → 2 | Admin | Không ban chính mình | Revoke REFRESH | `ID/modules/user/user.service.ts > adminBanUser()` |
| 2 → 1 | — | Không có API gỡ ban. Tài khoản tổ chức bị ban khi đang ở 3 vẫn có thể chuyển về 1 qua kích hoạt (99) | — | — |
| → xoá mềm | **User đăng nhập bất kỳ** (`DELETE /users/:id`) | — | Không revoke token | `user.service.ts > deleteUser()` |

## 11. Đơn đổi quà (`gift_redemptions.status`)

```mermaid
stateDiagram-v2
  [*] --> PROCESSING: user đổi quà
  PROCESSING --> SHIPPED
  PROCESSING --> CANCELLED: hoàn SP
  SHIPPED --> DELIVERED
  SHIPPED --> CANCELLED: hoàn SP
  DELIVERED --> [*]
  CANCELLED --> [*]
```

| Từ → sang | Ai (thực tế trong code) | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → PROCESSING | User đăng nhập | BR-222..BR-225 | Trừ tồn kho; trừ SP theo FIFO; ghi sổ âm | `RW/modules/gift/gift.service.ts > redeem()` |
| PROCESSING → SHIPPED; SHIPPED → DELIVERED | **User đăng nhập bất kỳ** (thiếu kiểm tra admin) | BR-230 | statusUpdatedAt | `updateRedemptionStatus()` |
| PROCESSING / SHIPPED → CANCELLED | như trên | BR-230 | cancelledAt; hoàn SP (lô mới); ghi GIFT_REDEEM_REFUND; không hoàn tồn kho | `updateRedemptionStatus()` |

Không có thông báo nào được gửi khi đơn đổi quà đổi trạng thái.

## 12. Season (`seasons.status`)

| Từ → sang | Ai | Điều kiện | Side effect | File |
|---|---|---|---|---|
| (mới) → ACTIVE 1 hoặc INACTIVE 2 | Admin | BR-247 | — | `season.service.ts > createSeason()` |
| ACTIVE → INACTIVE (finalize) | Admin | BR-248 | Snapshot bảng xếp hạng; chi SP theo payout tier; nếu `openNext` thì mở season mới | `finalizeSeason()` |
| ACTIVE ↔ INACTIVE (PATCH) | Admin | Không có điều kiện | **Không** snapshot, không chi SP | `patchSeason()` |

## 13. Background job, outbox, notification job (hạ tầng)

```mermaid
stateDiagram-v2
  [*] --> PENDING_12: createJob
  PENDING_12 --> FAILED_23: gửi SQS lỗi
  PENDING_12 --> INPROCESS_22: worker nhận
  INPROCESS_22 --> COMPLETED_17: xử lý OK
  INPROCESS_22 --> PENDING_12: lỗi, còn lượt (backoff)
  INPROCESS_22 --> FAILED_23: hết 5 lượt / jobType lạ
  PENDING_12 --> CANCELED_11: huỷ (chỉ incident)
```

- Áp dụng cho `background_jobs` (incident), `notification_jobs`, `reward_background_jobs` (`ecolink-server/shared/da2-queue/src/core/queue-worker.ts`, các `*/queue/background-job-store.ts`).
- **Outbox** (`outbox_events`, `INC/outbox/outbox-relay.ts`): 12 → 22 (claim bằng `FOR UPDATE SKIP LOCKED`) → 17 (publish OK) / 12 (retry, `min(900s, 30s·2^(a-1))`) / 23 (sau 10 lần). Nếu process crash sau khi claim, dòng bị kẹt ở 22. Circuit breaker của relay: 5 lỗi thì OPEN 30s.
- Queue `reward-intake` dùng NoopStore: không có dòng trạng thái nào.

## 14. Trạng thái nhị phân khác

| Entity | Trạng thái | Chuyển | File |
|---|---|---|---|
| Vote.value | 0 / 1 / -1 | Toggle (BR-130) | `INC/modules/vote/vote.service.ts` |
| SavedResource | active / deleted | Toggle (BR-133) | `saved_resource.service.ts` |
| CampaignCompletionVerification.value | 0 / 1 / -1 | Gửi lại cùng giá trị thì thành 0 | `campaign_completion_verification.service.ts` |
| CampaignAttendanceCheckIn | chưa có / có | Chỉ thêm, idempotent | `campaign_attendance.service.ts` |
| Notification.readAt | null → thời điểm | Chỉ một chiều | `NS/modules/notification/notification.service.ts > markRead()` |
| AuthToken | active → revoked / used / hết hạn | Xem BR-008..BR-012 | `ID/modules/auth/*` |
| OTP đơn tổ chức | active → used / expired | Xem BR-053, BR-054 | `organization-application-otp.service.ts` |
| Gift.isActive, BadgeDefinition.isActive / publishedAt | bật / tắt | Admin | `RW/modules/*` |
