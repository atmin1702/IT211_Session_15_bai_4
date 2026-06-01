## Phần 1 - Phân tích logic: Kiến trúc bảo mật & Mã hóa

### 1. Vai trò của các thành phần lõi
* **UserDetailsService:** Đóng vai trò là "cầu nối" truy xuất dữ liệu. Nhiệm vụ của nó là nhận vào `username`, truy vấn cơ sở dữ liệu để lấy thông tin chi tiết (bao gồm username, mật khẩu đã mã hóa, và danh sách các Roles), sau đó đóng gói thành đối tượng `UserDetails` cho Spring Security xử lý.
* **PasswordEncoder:** Chịu trách nhiệm đảm bảo an toàn cho mật khẩu thông qua hai hàm chính:
    * `encode()`: Mã hóa mật khẩu gốc trước khi lưu vào database.
    * `matches()`: So sánh mật khẩu người dùng vừa nhập với chuỗi mã hóa lấy từ database để xác định tính hợp lệ.

### 2. Mối nguy hiểm của mật khẩu 'Plain text' và NoOpPasswordEncoder
Việc lưu trữ mật khẩu không mã hóa (plain text) hoặc dùng `NoOpPasswordEncoder` là lỗ hổng chí mạng, ngay cả khi database có lớp phòng thủ mạnh:
* **Mối đe dọa nội bộ (Insider Threat):** DBA hoặc lập trình viên có quyền truy cập vào database có thể đọc trực tiếp mật khẩu của mọi người dùng.
* **Rủi ro lộ lọt (Data Breach):** Nếu hệ thống bị tấn công (ví dụ: qua SQL Injection) và hacker lấy được bản sao database, toàn bộ tài khoản sẽ bị chiếm đoạt ngay lập tức mà không cần tốn công giải mã.
* **Tấn công nhồi thông tin (Credential Stuffing):** Do người dùng hay dùng chung mật khẩu, hacker có thể dùng mật khẩu lộ lọt từ hệ thống thư viện để tấn công vào các tài khoản quan trọng khác của nạn nhân (email, ngân hàng).

### 3. Ưu điểm của BCryptPasswordEncoder
BCrypt là tiêu chuẩn vàng cho các ứng dụng web hiện đại nhờ hai cơ chế cốt lõi:
* **Tự động thêm Salt:** BCrypt tự động sinh ra một chuỗi ngẫu nhiên (salt) cộng vào mật khẩu trước khi băm. Cùng một mật khẩu "123456" sẽ sinh ra các chuỗi băm hoàn toàn khác nhau cho các tài khoản khác nhau, vô hiệu hóa các cuộc tấn công bằng từ điển (Dictionary Attack).
* **Độ trễ có chủ đích (Work Factor):** Thuật toán ép hệ thống phải tiêu tốn tài nguyên CPU và thời gian để tính toán (có thể tùy chỉnh độ khó). Điều này khiến các cuộc tấn công dò mật khẩu hàng loạt (Brute-force) trở nên bất khả thi về mặt thời gian và chi phí.

---

## Phần 2 - Thực thi: Phác thảo kiến trúc giải pháp

Dưới đây là sơ đồ luồng dữ liệu mô tả cách Spring Security tương tác với `CustomUserDetailsService` và cơ sở dữ liệu khi có yêu cầu đăng nhập.

```mermaid
graph TD
    A[Client / Trình duyệt] -->|1. Gửi form đăng nhập Username + Raw Password| B(Authentication Filter)
    
    subgraph Spring Security Context
        B -->|2. Ủy quyền xác thực| C{Authentication Provider}
        
        C -->|3. Gọi hàm loadUserByUsername| D[CustomUserDetailsService]
        
        D -->|4. Truy vấn bằng Username| E[(Database / MySQL)]
        E -->|5. Trả về Hashed Pwd & Roles| D
        
        D -->|6. Đóng gói thành UserDetails| C
        
        C -->|7. Gọi matches(Raw Pwd, Hashed Pwd)| F[BCryptPasswordEncoder]
        F -->|8. Trả về kết quả: True/False| C
    end
    
    C -->|9. Cập nhật SecurityContextHolder| B
    B -->|10. Xác thực Thành công: Cấp Session/Token\nXác thực Thất bại: Lỗi 401| A
```