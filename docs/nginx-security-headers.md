# Bài tập: Cấu hình HTTP Security Headers trên Nginx

> Đây là hướng dẫn và cấu hình mẫu trong workspace. Chưa có file hệ thống nào được sửa, Nginx chưa được reload và các response thực tế chưa được kiểm tra.

## 1. Cấu hình hoàn chỉnh

Mẫu tương ứng nằm ở [nginx/security-headers.conf.example](../nginx/security-headers.conf.example). Bản dưới đây giữ nguyên virtual host HTTPS và proxy hiện có, thêm ba security headers ở phạm vi `server`:

```nginx
server {
    listen 443 ssl;
    server_name secure.example.com;

    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://localhost:8080;
    }
}
```

Đường dẫn certificate/key ở đây được giữ nguyên từ tình huống; cần xác nhận chúng tồn tại, khớp nhau và hợp lệ trên VPS thật. Không coi file `.example` trong workspace là cấu hình hệ thống đã cài.

## 2. Ý nghĩa các header

- **`X-Frame-Options: SAMEORIGIN`**: trình duyệt chỉ cho trang được nhúng trong frame/iframe cùng origin. Điều này giảm nguy cơ Clickjacking từ một trang khác. Nếu ứng dụng hợp lệ cần được nhúng từ origin ngoài, phải thiết kế policy tương ứng; không đổi thành `ALLOW-FROM` như một giải pháp tổng quát.
- **`X-Content-Type-Options: nosniff`**: yêu cầu trình duyệt không tự suy đoán MIME type trong các trường hợp được quy định. Backend phải gửi `Content-Type` đúng cho HTML, JavaScript, CSS, ảnh và nội dung khác; `nosniff` không thay thế việc sửa MIME type sai.
- **`Strict-Transport-Security` (HSTS)**: sau khi trình duyệt nhận header qua HTTPS, trình duyệt sẽ dùng HTTPS cho domain trong khoảng `max-age=31536000` giây (một năm). `includeSubDomains` mở rộng chính sách tới toàn bộ subdomain.

`includeSubDomains` có phạm vi rộng: chỉ bật khi mọi subdomain đang dùng và dự kiến dùng đều có HTTPS hoạt động ổn định. Nếu có subdomain chỉ hỗ trợ HTTP, trình duyệt đã ghi nhận HSTS có thể không truy cập được subdomain đó. HSTS cũng không tự mã hóa response hoặc thay thế certificate; nó ra lệnh cho trình duyệt dùng HTTPS ở các lần truy cập tiếp theo. Hãy xác nhận HTTPS ổn định trước khi phát hành max-age dài; trình duyệt đã lưu policy sẽ giữ nó tới khi hết hạn.

## 3. Tham số `always` và kế thừa `add_header`

Mẫu dùng dạng:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

Tham số `always` yêu cầu Nginx thêm header bất kể status code response, kể cả nhiều response lỗi. Nếu bỏ `always`, `add_header` chỉ áp dụng cho nhóm status mà Nginx quy định; vì vậy header có thể xuất hiện ở response thành công nhưng vắng mặt khi lỗi.

Các header được đặt trong `server`, để áp dụng cho các `location` bên dưới nếu chúng không khai báo `add_header` riêng. Theo quy tắc kế thừa thông thường của Nginx, khi một `location` tự có bất kỳ `add_header` nào, các `add_header` ở cấp cha sẽ không được kế thừa tại location đó. Nếu header thiếu chỉ ở một route, kiểm tra `location` đó có `add_header` riêng không; có thể đặt đủ các header tại location hoặc dùng snippet include thống nhất. Không thêm `add_header` tùy tiện vào một location vì có thể làm mất các header kế thừa khác.

## 4. Kiểm tra và áp dụng an toàn

Các lệnh sau chỉ là hướng dẫn để bạn tự chạy trên VPS. Không sửa file hệ thống hoặc chạy `sudo` từ workspace này. Lệnh `sudo` có thể yêu cầu mật khẩu quản trị.

1. Xem cấu hình đang có, xác định server block thực sự nhận `secure.example.com`, các file include và nơi certificate được cấu hình:

   ```bash
   sudo nginx -T
   ```

2. Sao lưu file đang được include trước khi chỉnh. Nếu `sites-enabled` là symlink, xác định và sao lưu file nguồn phù hợp. Chỉ thêm directives vào server block HTTPS hiện có; không ghi đè toàn bộ `nginx.conf` và tránh tạo server block trùng `server_name`.
3. Kiểm tra cú pháp:

   ```bash
   sudo nginx -t
   ```

   Chỉ khi có kết quả `syntax is ok` và `test is successful` mới tiếp tục. Nếu lỗi, không reload; sửa lỗi được báo và chạy lại test.

4. Sau khi test thành công, áp dụng cấu hình:

   ```bash
   sudo systemctl reload nginx
   sudo systemctl status nginx
   ```

## 5. Kiểm tra response

Sau reload, chạy lệnh yêu cầu:

```bash
curl -I https://secure.example.com
```

Trong response headers, kiểm tra các giá trị:

```text
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

`curl -I` gửi HEAD request. Nếu ứng dụng/backend xử lý HEAD khác GET, có thể so sánh bằng GET không lưu body:

```bash
curl -sS -D - -o /dev/null https://secure.example.com/
```

Chỉ kết luận header hoạt động khi thấy nó trong response thực tế từ đúng hostname/server block; việc có directive trong file chưa chứng minh Nginx đã nạp hoặc response đã được trả như mong đợi.

## 6. Nếu header không xuất hiện

- Xác nhận DNS/hostname đang tới đúng Nginx và request chọn đúng server block (`server_name`, cổng `443`, certificate).
- Dùng `sudo nginx -T` để xem file nào thực sự được include; kiểm tra `add_header` có nằm trong đúng server/location không.
- Kiểm tra `location` có `add_header` riêng làm thay đổi quy tắc kế thừa không.
- Kiểm tra response status; `always` bao phủ các status code, nhưng cấu hình phải được nạp và request phải tới đúng Nginx.
- Backend, CDN hoặc load balancer phía trước/sau có thể thêm, xóa hoặc ghi đè header. So sánh response trực tiếp từ Nginx với response qua đường public và đọc access/error log.
- Nếu TLS/certificate lỗi, `curl -I` có thể dừng trước khi nhận response; xử lý certificate/TLS riêng thay vì tắt kiểm tra TLS của client.

Không tuyên bố cấu hình hoạt động nếu chưa quan sát response. Workspace hiện chỉ chứa ví dụ; không có kiểm tra VPS nào được thực hiện.
