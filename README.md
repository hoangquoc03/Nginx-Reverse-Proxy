# Bài 1: Cấu hình Nginx Reverse Proxy cơ bản

## 1. Mục tiêu bài học

Sau bài thực hành, bạn có thể cấu hình Nginx nhận HTTP trên cổng `80`, chuyển tiếp request tới ứng dụng React ở `127.0.0.1:3000`, hiểu các proxy header cơ bản và kiểm tra những lỗi thường gặp.

**Phạm vi:** `domain.com` là domain ví dụ. Khi triển khai, thay bằng domain thật đã trỏ DNS về IP VPS. Tài liệu này nằm trong workspace; nó **không** phải file cấu hình hệ thống và chưa cài đặt hay thay đổi gì trên VPS.

## 2. Mô hình hoạt động của Reverse Proxy

```text
Trình duyệt
  | HTTP: http://domain.com (cổng 80)
  v
Nginx trên VPS
  | HTTP nội bộ: http://127.0.0.1:3000
  v
Ứng dụng React
```

Reverse proxy là máy chủ trung gian đứng trước ứng dụng. Trình duyệt gửi request đến domain/Nginx; Nginx chuyển request đến upstream (ở đây là ứng dụng React), nhận phản hồi rồi trả lại cho trình duyệt. Người dùng không cần nhập `:3000`.

Nginx không tự tạo DNS. Domain thật cần có bản ghi DNS trỏ về VPS, và firewall/security group cần cho phép HTTP cổng `80`.

## 3. File cấu hình Nginx hoàn chỉnh

Server block mẫu dưới đây dành cho `/etc/nginx/sites-available/default` trên Ubuntu. Thay `domain.com` bằng domain thật; chỉ giữ `www.domain.com` nếu bạn có cấu hình DNS cho tên đó.

```nginx
server {
    # Nhận kết nối HTTP trên cổng 80.
    listen 80;

    # Các hostname mà server block này xử lý.
    server_name domain.com www.domain.com;

    location / {
        # Chuyển tiếp mọi đường dẫn tới ứng dụng React trên cùng VPS.
        proxy_pass http://127.0.0.1:3000;

        # Giữ thông tin host và địa chỉ/kiểu kết nối ban đầu.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Đây là **nội dung tham khảo để bạn triển khai trên VPS**, không phải file `/etc/nginx/sites-available/default` đã được tạo trong workspace. Không dán đè toàn bộ file trên VPS trước khi đọc cấu hình hiện có. Nếu file đã có server block hoặc cấu hình khác, hãy bảo toàn phần cần dùng, chỉnh đúng block và xử lý các block trùng `listen`/`server_name` cẩn thận.

## 4. Giải thích từng chỉ thị

1. **Reverse Proxy là gì?** Nginx nhận request thay ứng dụng, chuyển tiếp request đến ứng dụng phía sau và trả response về client. Ứng dụng không cần trực tiếp mở cổng riêng ra Internet.
2. **Khi truy cập `http://domain.com`:** DNS phân giải domain về IP VPS; trình duyệt kết nối cổng `80`; Nginx chọn server block theo địa chỉ nhận và `server_name`; `location /` khớp đường dẫn; Nginx gửi request đến `127.0.0.1:3000`; sau đó trả response upstream về trình duyệt.
3. **`listen 80;`** Nginx lắng nghe kết nối HTTP trên cổng TCP `80`. Đây là HTTP, không phải HTTPS.
4. **`server_name`** Khai báo hostname mà server block phục vụ. Nginx dùng giá trị host trong request để chọn block phù hợp. Domain phải trỏ DNS về VPS; nếu có `www`, DNS và cấu hình cũng phải có tên đó.
5. **`location /`** Khớp các request có đường dẫn bắt đầu bằng `/`, tức hầu hết đường dẫn thông thường như `/`, `/products` và `/assets/app.js`.
6. **`proxy_pass http://127.0.0.1:3000;`** Gửi request từ Nginx tới dịch vụ HTTP đang lắng nghe ở cổng `3000` trên chính VPS. Nginx là bên kết nối tới upstream; trình duyệt vẫn nói chuyện với domain và cổng `80`.
7. **Vì sao dùng `127.0.0.1`?** Đây là loopback IPv4 của chính máy chủ, nên upstream không cần được truy cập trực tiếp từ Internet. Điều này giảm bề mặt phơi bày cổng ứng dụng; cổng `3000` vẫn cần được bảo vệ bằng cấu hình bind/firewall thích hợp. `localhost` thường cũng là máy hiện tại nhưng có thể phân giải sang IPv6 (`::1`); hãy bảo đảm ứng dụng lắng nghe đúng địa chỉ mà Nginx dùng. Không mở cổng `3000` công khai nếu không có nhu cầu rõ ràng.
8. **Các header:**
   - `Host $host`: chuyển hostname người dùng yêu cầu tới ứng dụng.
   - `X-Real-IP $remote_addr`: chuyển IP mà Nginx thấy từ client.
   - `X-Forwarded-For $proxy_add_x_forwarded_for`: giữ chuỗi IP chuyển tiếp và thêm IP client; hữu ích khi có nhiều proxy. Chỉ tin header này từ proxy đáng tin cậy.
   - `X-Forwarded-Proto $scheme`: báo giao thức client dùng để đến Nginx (`http` trong cấu hình hiện tại; cấu hình HTTPS cần được thiết kế riêng).
9. **Reverse Proxy khác Redirect:** Reverse proxy chuyển tiếp phía máy chủ, URL trên trình duyệt vẫn là `http://domain.com`. Redirect gửi phản hồi chuyển hướng (ví dụ `301`/`302`) để trình duyệt tự gửi request đến URL khác; URL trình duyệt thay đổi và có thể làm lộ địa chỉ/cổng đích.

## 5. Các bước triển khai trên VPS

Các lệnh dưới đây là **hướng dẫn để bạn tự chạy trên VPS Ubuntu**, chưa được thực hiện trong workspace. Lệnh có `sudo` có thể yêu cầu mật khẩu quản trị; khi nhập mật khẩu, terminal thường không hiển thị ký tự.

1. **Kiểm tra ứng dụng tại cổng 3000.** Chạy trên VPS:

   ```bash
   curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:3000/
   sudo ss -ltnp | grep ':3000'
   ```

   Lệnh `curl` cho biết có nhận được phản hồi HTTP không; lệnh `ss` kiểm tra cổng đang lắng nghe. Không thấy listener hoặc bị `Connection refused` thường có nghĩa ứng dụng chưa chạy, chạy cổng khác, hoặc bind vào địa chỉ khác.

2. **Kiểm tra Nginx đã cài chưa.**

   ```bash
   nginx -v
   systemctl is-active nginx
   ```

   Nếu chưa cài, chỉ khi bạn chủ động muốn cài trên Ubuntu mới chạy `sudo apt update` rồi `sudo apt install nginx`. Nếu service chưa active, kiểm tra nguyên nhân trước khi tiếp tục.

3. **Đọc và sao lưu cấu hình hiện có trước khi sửa.**

   ```bash
   sudo sed -n '1,240p' /etc/nginx/sites-available/default
   ```

   Nếu file tồn tại, sao lưu bằng lệnh sau trước khi chỉnh:

   ```bash
   sudo cp -a /etc/nginx/sites-available/default "/etc/nginx/sites-available/default.bak.$(date +%Y%m%d-%H%M%S)"
   ```

   Mở để chỉnh thủ công:

   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```

   Đối chiếu server block ở mục 3. Không xóa cấu hình khác khi chưa hiểu tác dụng; thay domain ví dụ bằng domain thật. Nếu file chưa tồn tại, có thể tạo bằng editor sau khi xác nhận đường dẫn và cách site được kích hoạt trên máy đó.

4. **Kiểm tra cú pháp trước khi reload.**

   ```bash
   sudo nginx -t
   ```

   Chỉ tiếp tục nếu kết quả báo cấu hình hợp lệ (`syntax is ok` và `test is successful`). Nếu lỗi, đọc file/dòng được chỉ ra, sửa rồi chạy lại lệnh kiểm tra.

5. **Reload Nginx sau khi kiểm tra thành công.**

   ```bash
   sudo systemctl reload nginx
   ```

   Reload áp dụng cấu hình mới mà không yêu cầu dừng hẳn dịch vụ. Không chạy nếu `nginx -t` thất bại.

6. **Kiểm tra trạng thái dịch vụ.**

   ```bash
   sudo systemctl status nginx
   ```

7. **Kiểm tra bằng domain thật.** Đảm bảo DNS đã trỏ đúng IP VPS và firewall cho phép cổng `80`, rồi chạy từ máy có thể truy cập domain:

   ```bash
   curl -I http://domain.com
   ```

   Thay `domain.com` bằng domain thật. Có thể thử chọn server block tại máy VPS trước khi DNS hoạt động:

   ```bash
   curl -I -H 'Host: domain.com' http://127.0.0.1/
   ```

8. **Xem log khi gặp lỗi.**

   ```bash
   sudo tail -f /var/log/nginx/error.log
   sudo tail -f /var/log/nginx/access.log
   ```

   `error.log` ghi lỗi Nginx/upstream; `access.log` ghi các request và status code. Nhấn `Ctrl+C` để thoát chế độ theo dõi log.

## 6. Các bước kiểm thử

### Khi ứng dụng React đang chạy

1. Xác nhận ứng dụng trả lời tại `http://127.0.0.1:3000/` bằng lệnh ở bước triển khai 1.
2. Gửi request qua Nginx bằng `curl -I http://domain.com` (hoặc lệnh `Host` nội bộ nếu DNS chưa sẵn sàng).
3. Kết quả mong đợi là Nginx trả về status mà ứng dụng cung cấp, thường là `200 OK` cho trang hoạt động bình thường. Header/status thực tế phụ thuộc ứng dụng và cấu hình của nó.
4. Nếu không đạt, đối chiếu `error.log`, `access.log`, DNS, firewall, `server_name`, port và địa chỉ bind của ứng dụng.

### Khi ứng dụng bị dừng (chỉ kiểm thử nếu bạn chủ động)

**Không dừng ứng dụng đang chạy như một phần tự động của bài này.** Chỉ làm phép thử sau khi đã lên kế hoạch, biết cách khởi động lại đúng ứng dụng và chấp nhận gián đoạn tạm thời. Dừng/khởi động ứng dụng bằng process manager mà chính bạn dùng (ví dụ systemd, PM2, Docker); không dùng lệnh `kill` tùy tiện. Với systemd, mẫu lệnh là `sudo systemctl stop <ten-dich-vu-ung-dung>` và sau khi kiểm tra `sudo systemctl start <ten-dich-vu-ung-dung>`; thay placeholder bằng tên service thực tế. Nếu không biết service/process manager, bỏ qua phép thử này.

Khi upstream dừng, gửi lại request bằng `curl -I http://domain.com` và xem log. Nếu Nginx vẫn hoạt động nhưng không kết nối được tới upstream, kết quả thường gặp là `502 Bad Gateway`. Khởi động lại ứng dụng ngay sau phép thử và xác nhận request hoạt động trở lại.

## 7. Phân biệt HTTP 502, 504 và 500

| Mã | Ý nghĩa trong tình huống này | Ví dụ |
|---|---|---|
| `502 Bad Gateway` | Nginx không nhận được phản hồi upstream hợp lệ. Đây là lỗi thường gặp khi ứng dụng không chạy hoặc kết nối bị từ chối. | Không có listener tại `127.0.0.1:3000`. |
| `504 Gateway Timeout` | Nginx chờ upstream quá lâu mà không nhận được phản hồi kịp thời. | Ứng dụng/upstream phản hồi quá chậm hoặc bị treo. |
| `500 Internal Server Error` | Lỗi nội bộ của máy chủ hoặc ứng dụng; không phải mã mặc định chỉ vì React bị dừng. | Ứng dụng tự trả `500`, hoặc một xử lý nội bộ phát sinh lỗi. |

**Trả lời câu hỏi:** Nếu Nginx còn hoạt động nhưng không kết nối được ứng dụng React ở cổng `3000`, thông thường Nginx trả `502 Bad Gateway`. Không thể khẳng định mọi lỗi upstream luôn là cùng một mã; status còn tùy loại lỗi, thời gian chờ và cấu hình Nginx/ứng dụng.

## 8. Kết quả mong đợi

- Request đến domain trên cổng `80` được Nginx nhận.
- Nginx proxy request đến ứng dụng ở `127.0.0.1:3000` và giữ các thông tin client cần thiết qua header.
- Người dùng truy cập domain mà không cần thêm `:3000`.
- Khi upstream không sẵn sàng, Nginx thường báo `502`; log giúp xác định nguyên nhân.

## 9. Lỗi thường gặp và cách khắc phục

| Hiện tượng | Kiểm tra/cách khắc phục |
|---|---|
| `502 Bad Gateway` | Kiểm tra ứng dụng có chạy, đúng cổng và bind tại địa chỉ Nginx gọi hay không; xem `/var/log/nginx/error.log`. |
| `504 Gateway Timeout` | Kiểm tra ứng dụng có bị treo/chậm, tài nguyên VPS và log ứng dụng; chỉ tăng timeout khi đã xác định nhu cầu. |
| `nginx -t` báo lỗi | Sửa lỗi cú pháp hoặc dấu `;`/`{}` ở file và dòng được báo; chưa reload cho đến khi kiểm tra thành công. |
| Domain không truy cập được | Kiểm tra DNS trỏ về VPS, firewall/security group cho phép cổng `80`, Nginx đang chạy và `listen 80` đúng. |
| Trả nhầm website/server block | Kiểm tra `server_name`, các cấu hình site được enable và server block trùng; kiểm tra bằng request có header `Host`. |
| Nginx không kết nối được dù app chạy | Kiểm tra app có thực sự nghe trên `127.0.0.1:3000` không; `localhost` có thể resolve sang IPv6 trong một số môi trường. |
| Cổng 3000 truy cập trực tiếp được từ Internet | Rà soát địa chỉ bind và firewall; chỉ mở cổng cần thiết, thông thường client chỉ cần cổng `80` (hoặc `443` khi triển khai HTTPS). |

## 10. Kết luận ngắn để trình bày

“Nginx Reverse Proxy nhận HTTP trên cổng 80 theo domain, sau đó chuyển tiếp request tới ứng dụng React nội bộ ở `127.0.0.1:3000`. Các header proxy giúp ứng dụng biết host, IP client và giao thức ban đầu. Khi upstream không phản hồi hợp lệ, lỗi thường gặp là `502 Bad Gateway`; còn `504` biểu thị quá thời gian chờ và `500` là lỗi nội bộ. Trước khi áp dụng cấu hình, cần kiểm tra và sao lưu file hiện có, chạy `nginx -t`, rồi mới reload.”

## Trạng thái thực hiện

- **Đã thực hiện trong workspace:** tạo tài liệu hướng dẫn `README.md`.
- **Chưa thực hiện:** truy cập VPS, đọc/sửa `/etc/nginx/sites-available/default`, cài/reload Nginx, kiểm tra domain hoặc dừng ứng dụng. Các việc này cần bạn tự thực hiện trên VPS theo hướng dẫn ở trên.
- **Mã nguồn React:** workspace không có mã nguồn để kiểm tra; không có mã ứng dụng nào bị sửa.
