# Bài 1: Cấu hình Nginx Reverse Proxy cơ bản

## 1. Mục tiêu bài học

Sau bài thực hành, bạn có thể cấu hình Nginx nhận HTTP trên cổng `80`, chuyển tiếp request API tới API Gateway ở `127.0.0.1:8080` và request giao diện tới ứng dụng React ở `127.0.0.1:3000`, hiểu các proxy header cơ bản và kiểm tra những lỗi thường gặp.

**Phạm vi:** `domain.com` là domain ví dụ. Khi triển khai, thay bằng domain thật đã trỏ DNS về IP VPS. Tài liệu này nằm trong workspace; nó **không** phải file cấu hình hệ thống và chưa cài đặt hay thay đổi gì trên VPS.

## 2. Mô hình hoạt động của Reverse Proxy

```text
Trình duyệt
  | HTTP: http://domain.com (cổng 80)
  v
Nginx trên VPS
   |-- /api/... --> http://127.0.0.1:8080 --> Backend API Gateway
   `-- các đường dẫn khác --> http://127.0.0.1:3000 --> Frontend React
```

Reverse proxy là máy chủ trung gian đứng trước ứng dụng. Trình duyệt gửi request đến domain/Nginx; Nginx chọn upstream theo đường dẫn, nhận phản hồi rồi trả lại cho trình duyệt. Request `/api/...` đi tới API Gateway; các đường dẫn khác đi tới React. Người dùng không cần nhập `:3000` hoặc `:8080`.

Nginx không tự tạo DNS. Domain thật cần có bản ghi DNS trỏ về VPS, và firewall/security group cần cho phép HTTP cổng `80`.

## 3. File cấu hình Nginx hoàn chỉnh

Server block mẫu dưới đây dành cho `/etc/nginx/sites-available/default` trên Ubuntu. Thay `domain.com` bằng domain thật; chỉ giữ `www.domain.com` nếu bạn có cấu hình DNS cho tên đó.

```nginx
server {
    # Nhận kết nối HTTP trên cổng 80.
    listen 80;

    # Các hostname mà server block này xử lý.
    server_name domain.com www.domain.com;

   # API Gateway: xử lý các request có đường dẫn bắt đầu bằng /api/.
   location /api/ {
      proxy_pass http://127.0.0.1:8080;

      # Chuyển tiếp host và thông tin kết nối ban đầu.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

   # Frontend React: nhận các đường dẫn không khớp location /api/.
   location / {
      proxy_pass http://127.0.0.1:3000;

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
2. **Khi truy cập `http://domain.com`:** DNS phân giải domain về IP VPS; trình duyệt kết nối cổng `80`; Nginx chọn server block theo địa chỉ nhận và `server_name`. Với `/api/products`, Nginx chọn `location /api/` và gửi tới cổng `8080`; với `/` hoặc `/products`, Nginx chọn `location /` và gửi tới cổng `3000`. Nginx trả response tương ứng về trình duyệt.
3. **`listen 80;`** Nginx lắng nghe kết nối HTTP trên cổng TCP `80`. Đây là HTTP, không phải HTTPS.
4. **`server_name`** Khai báo hostname mà server block phục vụ. Nginx dùng giá trị host trong request để chọn block phù hợp. Domain phải trỏ DNS về VPS; nếu có `www`, DNS và cấu hình cũng phải có tên đó.
5. **`location /api/` và `location /`** `/api/` khớp các đường dẫn bắt đầu bằng tiền tố `/api/`; `/` là tiền tố tổng quát, nhận các request còn lại. Nginx chọn tiền tố phù hợp dài nhất, vì vậy `/api/orders` vào API Gateway thay vì React. Đường dẫn `/api` (không có dấu `/` cuối) không khớp `location /api/` và theo cấu hình mẫu sẽ vào frontend; hãy gọi endpoint dưới dạng `/api/...` hoặc cấu hình xử lý `/api` riêng nếu cần.
6. **`proxy_pass`** `proxy_pass http://127.0.0.1:8080;` chuyển request API tới API Gateway; `proxy_pass http://127.0.0.1:3000;` chuyển các request còn lại tới React. Do `proxy_pass` không có URI phía sau host/port, phần URI gốc được giữ lại: ví dụ `/api/orders` được chuyển tới upstream API dưới đường dẫn `/api/orders`. Nginx kết nối upstream; trình duyệt vẫn truy cập domain qua cổng `80`.
7. **Vì sao dùng `127.0.0.1`?** Đây là loopback IPv4 của chính máy chủ, nên upstream không cần được truy cập trực tiếp từ Internet. Điều này giảm bề mặt phơi bày các cổng ứng dụng (`3000` và `8080`); các cổng này vẫn cần được bảo vệ bằng cấu hình bind/firewall thích hợp. `localhost` có thể phân giải sang IPv6 (`::1`); hãy bảo đảm mỗi dịch vụ lắng nghe đúng địa chỉ mà Nginx gọi. Không mở các cổng ứng dụng công khai nếu không có nhu cầu rõ ràng.
8. **Các header:**
   - `Host $host`: chuyển hostname người dùng yêu cầu tới ứng dụng.
   - `X-Real-IP $remote_addr`: chuyển IP mà Nginx thấy từ client.
   - `X-Forwarded-For $proxy_add_x_forwarded_for`: giữ chuỗi IP chuyển tiếp và thêm IP client; hữu ích khi có nhiều proxy. Chỉ tin header này từ proxy đáng tin cậy.
   - `X-Forwarded-Proto $scheme`: báo giao thức client dùng để đến Nginx (`http` trong cấu hình hiện tại; cấu hình HTTPS cần được thiết kế riêng).
9. **Reverse Proxy khác Redirect:** Reverse proxy chuyển tiếp phía máy chủ, URL trên trình duyệt vẫn là `http://domain.com`. Redirect gửi phản hồi chuyển hướng (ví dụ `301`/`302`) để trình duyệt tự gửi request đến URL khác; URL trình duyệt thay đổi và có thể làm lộ địa chỉ/cổng đích.

## 5. Các bước triển khai trên VPS

Các lệnh dưới đây là **hướng dẫn để bạn tự chạy trên VPS Ubuntu**, chưa được thực hiện trong workspace. Lệnh có `sudo` có thể yêu cầu mật khẩu quản trị; khi nhập mật khẩu, terminal thường không hiển thị ký tự.

1. **Kiểm tra cả hai ứng dụng/upstream.** Chạy trên VPS:

   ```bash
   curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:3000/
   curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:8080/
   sudo ss -ltnp | grep -E ':(3000|8080)\b'
   ```

   `curl` cho biết mỗi upstream có phản hồi HTTP không; `ss` kiểm tra hai cổng đang lắng nghe. Endpoint gốc `/` của API có thể trả `404` nếu gateway không định nghĩa route này; điều đó vẫn cho thấy đã có phản hồi HTTP. Không thấy listener hoặc bị `Connection refused` thường có nghĩa dịch vụ chưa chạy, chạy cổng khác, hoặc bind vào địa chỉ khác.

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

1. Xác nhận frontend ở `http://127.0.0.1:3000/` và API Gateway ở `http://127.0.0.1:8080/` hoặc endpoint API hợp lệ.
2. Kiểm tra giao diện qua Nginx: `curl -I http://domain.com/` (hoặc dùng lệnh `Host` nội bộ nếu DNS chưa sẵn sàng). Request này phải được chuyển tới cổng `3000`.
3. Kiểm tra API qua Nginx: `curl -i http://domain.com/api/<duong-dan-api-thuc-te>`. Thay placeholder bằng endpoint có thật. Request này phải được chuyển tới cổng `8080`; mã phản hồi phụ thuộc API và endpoint.
4. Nếu không đạt, đối chiếu `error.log`, `access.log`, DNS, firewall, `server_name`, cả hai port và địa chỉ bind của các dịch vụ.

### Khi ứng dụng bị dừng (chỉ kiểm thử nếu bạn chủ động)

**Không dừng ứng dụng đang chạy như một phần tự động của bài này.** Chỉ làm phép thử sau khi đã lên kế hoạch, biết cách khởi động lại đúng ứng dụng và chấp nhận gián đoạn tạm thời. Dừng/khởi động ứng dụng bằng process manager mà chính bạn dùng (ví dụ systemd, PM2, Docker); không dùng lệnh `kill` tùy tiện. Với systemd, mẫu lệnh là `sudo systemctl stop <ten-dich-vu-ung-dung>` và sau khi kiểm tra `sudo systemctl start <ten-dich-vu-ung-dung>`; thay placeholder bằng tên service thực tế. Nếu không biết service/process manager, bỏ qua phép thử này.

Khi một upstream dừng, kiểm tra riêng route của nó: frontend bằng `curl -I http://domain.com/`; API bằng `curl -i http://domain.com/api/<duong-dan-api-thuc-te>`. Nếu Nginx vẫn hoạt động nhưng không kết nối được tới upstream tương ứng, kết quả thường gặp là `502 Bad Gateway`. Khởi động lại đúng dịch vụ ngay sau phép thử và xác nhận route đó hoạt động trở lại.

## 7. Phân biệt HTTP 502, 504 và 500

| Mã                          | Ý nghĩa trong tình huống này                                                                                           | Ví dụ                                                       |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `502 Bad Gateway`           | Nginx không nhận được phản hồi upstream hợp lệ. Đây là lỗi thường gặp khi một ứng dụng không chạy hoặc kết nối bị từ chối. | Không có listener tại `127.0.0.1:3000` hoặc `127.0.0.1:8080`. |
| `504 Gateway Timeout`       | Nginx chờ upstream quá lâu mà không nhận được phản hồi kịp thời.                                                       | Ứng dụng/upstream phản hồi quá chậm hoặc bị treo.           |
| `500 Internal Server Error` | Lỗi nội bộ của máy chủ hoặc ứng dụng; không phải mã mặc định chỉ vì một upstream bị dừng.                                  | Ứng dụng tự trả `500`, hoặc một xử lý nội bộ phát sinh lỗi. |

**Trả lời câu hỏi:** Nếu Nginx còn hoạt động nhưng không kết nối được upstream được chọn (React ở cổng `3000` hoặc API Gateway ở cổng `8080`), thông thường Nginx trả `502 Bad Gateway`. Không thể khẳng định mọi lỗi upstream luôn là cùng một mã; status còn tùy loại lỗi, thời gian chờ và cấu hình Nginx/ứng dụng.

## 8. Kết quả mong đợi

- Request đến domain trên cổng `80` được Nginx nhận.
- Nginx chuyển request `/api/...` đến `127.0.0.1:8080` và các đường dẫn còn lại đến `127.0.0.1:3000`, đồng thời giữ các thông tin client cần thiết qua header.
- Người dùng truy cập domain mà không cần thêm `:3000` hoặc `:8080`.
- Khi upstream không sẵn sàng, Nginx thường báo `502`; log giúp xác định nguyên nhân.

## 9. Lỗi thường gặp và cách khắc phục

| Hiện tượng                                    | Kiểm tra/cách khắc phục                                                                                                           |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `502 Bad Gateway`                             | Kiểm tra upstream tương ứng với URL: frontend cổng `3000`, API cổng `8080`; xác nhận service đang chạy/bind đúng địa chỉ và xem `/var/log/nginx/error.log`. |
| `504 Gateway Timeout`                         | Kiểm tra ứng dụng có bị treo/chậm, tài nguyên VPS và log ứng dụng; chỉ tăng timeout khi đã xác định nhu cầu.                      |
| `nginx -t` báo lỗi                            | Sửa lỗi cú pháp hoặc dấu `;`/`{}` ở file và dòng được báo; chưa reload cho đến khi kiểm tra thành công.                           |
| Domain không truy cập được                    | Kiểm tra DNS trỏ về VPS, firewall/security group cho phép cổng `80`, Nginx đang chạy và `listen 80` đúng.                         |
| Trả nhầm website/server block                 | Kiểm tra `server_name`, các cấu hình site được enable và server block trùng; kiểm tra bằng request có header `Host`.              |
| Nginx không kết nối được dù app chạy          | Kiểm tra dịch vụ tương ứng có thực sự nghe trên `127.0.0.1:3000` hoặc `127.0.0.1:8080` không; `localhost` có thể resolve sang IPv6. |
| Cổng 3000/8080 truy cập trực tiếp từ Internet | Rà soát địa chỉ bind và firewall; thông thường client chỉ cần cổng `80` (hoặc `443` khi triển khai HTTPS).                              |

## 10. Kết luận ngắn để trình bày

“Nginx Reverse Proxy nhận HTTP trên cổng 80 theo domain, định tuyến `/api/...` tới API Gateway nội bộ ở `127.0.0.1:8080` và các đường dẫn khác tới React ở `127.0.0.1:3000`. Các header proxy giúp upstream biết host, IP client và giao thức ban đầu. Khi upstream không phản hồi hợp lệ, lỗi thường gặp là `502 Bad Gateway`; còn `504` biểu thị quá thời gian chờ và `500` là lỗi nội bộ. Trước khi áp dụng cấu hình, cần kiểm tra và sao lưu file hiện có, chạy `nginx -t`, rồi mới reload.”

## Trạng thái thực hiện

- **Đã thực hiện trong workspace:** tạo tài liệu hướng dẫn `README.md`.
- **Chưa thực hiện:** truy cập VPS, đọc/sửa `/etc/nginx/sites-available/default`, cài/reload Nginx, kiểm tra domain hoặc dừng ứng dụng. Các việc này cần bạn tự thực hiện trên VPS theo hướng dẫn ở trên.
- **Mã nguồn React:** workspace không có mã nguồn để kiểm tra; không có mã ứng dụng nào bị sửa.
