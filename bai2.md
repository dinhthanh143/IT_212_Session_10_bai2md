# Bài 2: Tối Ưu Hóa Prompt Đặc Tả Yêu Cầu Chức Năng

## Prompt tối ưu (5 thành phần)

```text
=== VAI TRÒ ===
Bạn là một System Analyst cao cấp, chuyên gia về bảo mật web và kiến trúc Authentication/Authorization.

=== MỤC TIÊU ===
Viết phần Đặc tả Yêu cầu Chức năng (Functional Requirements) cho module Đăng nhập (Login) của hệ thống Shop AI, sử dụng chuẩn JWT (JSON Web Token) cho xác thực và RBAC (Role-Based Access Control) cho phân quyền.

=== NGỮ CẢNH ===
- Hệ thống là một nền tảng thương mại điện tử (Shop AI) có các vai trò: CUSTOMER (khách hàng), SELLER (người bán), ADMIN (quản trị viên).
- Module đăng nhập hiện tại sử dụng Spring Security + JWT (access token + refresh token).
- Token JWT có thời hạn 15 phút cho access token, 7 ngày cho refresh token.
- Mật khẩu được mã hóa bằng BCrypt.
- Hệ thống lưu trạng thái tài khoản (ACTIVE, INACTIVE, BANNED) trong bảng users.

=== RÀNG BUỘC KỸ THUẬT ===
- PHẢI đặc tả đầy đủ 3 trường hợp ngoại lệ bắt buộc sau đây:
    1. Người dùng nhập sai mật khẩu quá 5 lần: mô tả cơ chế khóa tạm thời (lockout), thời gian khóa, cách reset bộ đếm.
    2. JWT token hết hạn (expired) trong lúc phiên làm việc đang diễn ra: mô tả luồng refresh token tự động, HTTP status code trả về, hành vi của frontend khi nhận 401.
    3. Tài khoản người dùng ở trạng thái khóa/Inactive nhưng cố tình đăng nhập: mô tả thông báo lỗi cụ thể, hành vi hệ thống (từ chối hay cho phép), phân biệt với trường hợp sai mật khẩu.
- Mỗi yêu cầu chức năng PHẢI có mã định danh (FR-LOGIN-XX) duy nhất.
- Các điều kiện tiên quyết (Pre-condition) và ngoại lệ (Exception flow) PHẢI được định nghĩa rõ ràng cho từng use case.

=== ĐỊNH DẠNG ĐẦU RA ===
Trình bày dưới dạng bảng Markdown với các cột: Mã ID, Tên chức năng, Mô tả chi tiết (gồm Pre-condition, Main flow, Exception flow), Ưu tiên (High/Medium/Low).
```

---

## Tài liệu Functional Requirements do AI sinh ra

```text
# Functional Requirements - Module Đăng nhập (Shop AI)

| Mã ID | Tên chức năng | Mô tả chi tiết | Ưu tiên |
|-------|---------------|----------------|---------|
| FR-LOGIN-01 | Đăng nhập với thông tin hợp lệ | **Pre-condition:** Tài khoản ở trạng thái ACTIVE, chưa bị lock. **Main flow:** (1) User nhập email + password. (2) Hệ thống xác thực BCrypt. (3) Tạo access token (15 phút) + refresh token (7 ngày). (4) Trả về 200 OK kèm token. **Exception flow:** Sai email/password → 401 Unauthorized, tăng biến đếm login_attempts. | High |
| FR-LOGIN-02 | Khóa tài khoản tạm thời khi sai mật khẩu quá 5 lần | **Pre-condition:** Biến đếm login_attempts >= 5 trong vòng 15 phút. **Main flow:** (1) Hệ thống khóa tài khoản tạm thời (LOCKED_TEMPORARY) trong 30 phút. (2) Ghi log sự kiện với timestamp. (3) Trả về 423 Locked kèm message "Tài khoản bị khóa tạm thời do nhập sai mật khẩu quá nhiều lần. Vui lòng thử lại sau 30 phút." **Exception flow:** Trong thời gian khóa, nếu tiếp tục đăng nhập → vẫn trả về 423, không tăng login_attempts. Sau 30 phút, tự động reset login_attempts về 0 và chuyển trạng thái về ACTIVE. | High |
| FR-LOGIN-03 | Xử lý JWT access token hết hạn (expired) | **Pre-condition:** Request gửi kèm access token đã hết hạn (expired). **Main flow:** (1) JwtAuthenticationFilter phát hiện token hết hạn. (2) Trả về HTTP 401 Expired Token kèm mã lỗi "TOKEN_EXPIRED". (3) Frontend nhận 401 → tự động gọi endpoint /api/v1/auth/refresh với refresh token. (4) Hệ thống kiểm tra refresh token còn hạn → cấp access token mới. (5) Frontend retry request gốc với token mới. **Exception flow:** Refresh token cũng hết hạn → 401 với mã "SESSION_EXPIRED", buộc user đăng nhập lại. Nếu refresh token bị thu hồi (revoked) → 401 Forbidden. | High |
| FR-LOGIN-04 | Tài khoản Inactive cố tình đăng nhập | **Pre-condition:** Tài khoản tồn tại nhưng status = INACTIVE hoặc BANNED. **Main flow:** (1) Hệ thống kiểm tra status tại bước pre-authentication. (2) Từ chối đăng nhập ngay lập tức, KHÔNG kiểm tra mật khẩu (tránh leak thông tin). (3) Trả về HTTP 403 Forbidden với message "Tài khoản đã bị vô hiệu hóa. Vui lòng liên hệ quản trị viên." **Exception flow:** Không có ngoại lệ cho flow này. Ghi log sự kiện đăng nhập bất hợp pháp (unauthorized login attempt on inactive account) vào hệ thống audit. | High |
| FR-LOGIN-05 | Phân quyền RBAC sau đăng nhập | **Pre-condition:** User đã có JWT token hợp lệ. **Main flow:** (1) JWT chứa claim "role" (CUSTOMER/SELLER/ADMIN). (2) Mỗi request được JwtAuthenticationFilter kiểm tra role. (3) Nếu role phù hợp với @PreAuthorize của endpoint → cho phép truy cập. **Exception flow:** Role không đủ quyền → 403 Forbidden. Token không chứa role claim → 401 Unauthorized. | High |
| FR-LOGIN-06 | Đăng xuất (Logout) | **Pre-condition:** User đang có phiên làm việc hợp lệ. **Main flow:** (1) User gọi endpoint /api/v1/auth/logout. (2) Hệ thống thu hồi refresh token (thêm vào blacklist/revoke). (3) Xóa bộ nhớ đệm token phía frontend. **Exception flow:** Token không hợp lệ → 400 Bad Request. Token đã hết hạn → vẫn thực hiện xóa token phía client (idempotent). | Medium |
```
