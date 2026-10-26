# Bài tập về nhà: Triển khai WAF trên Nginx bảo vệ Web Application

## 1. Giải thích cấu hình Docker Compose
Hệ thống được triển khai thông qua Docker Compose với 2 services chính: `web` (Juice Shop) và `waf` (Nginx + ModSecurity). 

Các biến môi trường quan trọng được cấu hình trong service `waf`:
*   **`BACKEND: http://web:3000`**: Định tuyến cho WAF. WAF sẽ nhận các request từ người dùng và chuyển tiếp các request hợp lệ tới địa chỉ ứng dụng web thực sự đang chạy ở port 3000.
*   **`MODSEC_RULE_ENGINE: "On"`**: Bật chế độ phòng thủ chủ động của ModSecurity. Ở chế độ này, WAF sẽ trực tiếp ngăn chặn các request độc hại. Nếu để `DetectionOnly`, hệ thống chỉ ghi log cảnh báo mà không chặn.
*   **`BLOCKING_PARANOIA: "1"`**: Cấu hình mức độ nhạy cảm của WAF. Mức 1 là mức cơ bản, giúp chặn các cuộc tấn công phổ biến mà ít gây ra cảnh báo nhầm. Càng tăng lên cao, WAF kiểm tra càng gắt gao nhưng dễ chặn nhầm request hợp lệ.

*Lưu ý về Port:* 
*   **Cổng 8080**: Cổng của WAF. Traffic đi qua đây sẽ được kiểm tra an toàn.
*   **Cổng 3000**: Cổng của ứng dụng gốc. Truy cập thẳng vào đây sẽ bỏ qua lớp bảo vệ của WAF.

## 2. Các bước triển khai
1. Tạo file `docker-compose.yml` với nội dung cấu hình 2 services.
2. Mở terminal tại thư mục chứa file, chạy lệnh khởi động nền:
   ```bash
   docker compose up -d
   ```
3. Truy cập trình duyệt để kiểm tra:
   * `http://localhost:3000`: Ứng dụng Juice Shop chạy trực tiếp.
   * `http://localhost:8080`: Ứng dụng Juice Shop được bảo vệ qua WAF.

## 3. Kết quả kiểm thử tấn công

Để kiểm chứng hiệu quả của WAF, hệ thống đã được thử nghiệm với payload tấn công **Path Traversal** (`../../../../etc/passwd`).

*   **Trường hợp 1: Tấn công trực tiếp vào Web Server**
    *   **Lệnh:** `curl.exe -I "http://localhost:3000/?file=../../../../etc/passwd"`
    *   **Kết quả:** Server trả về mã HTTP `200 OK`. 
    *   **Kết luận:** Payload độc hại đã lọt qua thành công vì không có lớp bảo vệ, ứng dụng web có thể bị khai thác.

*   **Trường hợp 2: Tấn công qua WAF**
    *   **Lệnh:** `curl.exe -I "http://localhost:8080/?file=../../../../etc/passwd"`
    *   **Kết quả:** Server trả về mã HTTP `403 Forbidden`.
    *   **Kết luận:** ModSecurity đã đối chiếu chuỗi độc hại `../../../` với bộ luật OWASP CRS, phát hiện dấu hiệu bất thường và chủ động chặn đứng request trước khi nó kịp tiếp cận ứng dụng web.