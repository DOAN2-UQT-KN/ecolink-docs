# ecolink-docs

Tài liệu kỹ thuật và nghiệp vụ của nền tảng **Ecolink** — người dân báo cáo điểm rác/ô nhiễm, tổ chức và tình nguyện viên chạy chiến dịch dọn dẹp, tích điểm xanh đổi quà.

Toàn bộ nội dung được viết dựa trên source code của [`ecolink-server`](https://github.com/DOAN2-UQT-KN/ecolink-server) và [`ecolink-client`](https://github.com/DOAN2-UQT-KN/ecolink-client). Đường dẫn trong tài liệu tính từ thư mục workspace chứa các repo đó (`ecolink/`), ví dụ `ecolink-server/services/incident-service/...`.

## Mục lục

| File | Nội dung |
|---|---|
| [00-overview.md](00-overview.md) | Tổng quan, kiến trúc, giao tiếp, cấu hình |
| [01-data-model.md](01-data-model.md) | ERD và từ điển dữ liệu |
| [02-business-flows.md](02-business-flows.md) | Luồng nghiệp vụ và sequence diagram |
| [03-business-rules.md](03-business-rules.md) | Bảng business rule BR-xxx |
| [04-state-machines.md](04-state-machines.md) | Vòng đời trạng thái của các entity |
| [05-permissions.md](05-permissions.md) | Xác thực và ma trận phân quyền |
| [06-frontend.md](06-frontend.md) | Web client: route, màn hình → API, state, xử lý lỗi |
| [BUSINESS-OVERVIEW.md](BUSINESS-OVERVIEW.md) | Bản tổng hợp cho người không làm kỹ thuật |
| [ORG_CREATION_FLOW.md](ORG_CREATION_FLOW.md) | Luồng tạo tổ chức (as-is) |
| [REFACTOR_ORG_CREATION_FLOW.md](REFACTOR_ORG_CREATION_FLOW.md) | Luồng xác thực tổ chức & Blue Tick (to-be) |
| [services/](services/) | Tài liệu từng service: api-gateway, identity, incident, notification, reward, translation-worker, ai |

> `99-open-issues.md` (vấn đề mở, nghi bug, lỗ hổng) chỉ lưu nội bộ và không có trong repo này — các liên kết tới file đó sẽ không mở được.

## Quy tắc cập nhật

Docs phải đi cùng thay đổi code: sửa tính năng, endpoint, schema, trạng thái, quyền hay màn hình thì cập nhật file tương ứng trong cùng đợt làm việc. Chỉ ghi điều code thể hiện, giữ nguyên format (bảng, trích dẫn `path > function()`, mermaid, mã BR-xxx). Không đưa secret, giá trị env thật hay chi tiết lỗ hổng vào đây.
