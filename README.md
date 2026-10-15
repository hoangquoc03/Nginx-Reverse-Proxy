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

| Mã                          | Ý nghĩa trong tình huống này                                                                                               | Ví dụ                                                         |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `502 Bad Gateway`           | Nginx không nhận được phản hồi upstream hợp lệ. Đây là lỗi thường gặp khi một ứng dụng không chạy hoặc kết nối bị từ chối. | Không có listener tại `127.0.0.1:3000` hoặc `127.0.0.1:8080`. |
| `504 Gateway Timeout`       | Nginx chờ upstream quá lâu mà không nhận được phản hồi kịp thời.                                                           | Ứng dụng/upstream phản hồi quá chậm hoặc bị treo.             |
| `500 Internal Server Error` | Lỗi nội bộ của máy chủ hoặc ứng dụng; không phải mã mặc định chỉ vì một upstream bị dừng.                                  | Ứng dụng tự trả `500`, hoặc một xử lý nội bộ phát sinh lỗi.   |

**Trả lời câu hỏi:** Nếu Nginx còn hoạt động nhưng không kết nối được upstream được chọn (React ở cổng `3000` hoặc API Gateway ở cổng `8080`), thông thường Nginx trả `502 Bad Gateway`. Không thể khẳng định mọi lỗi upstream luôn là cùng một mã; status còn tùy loại lỗi, thời gian chờ và cấu hình Nginx/ứng dụng.

## 8. Kết quả mong đợi

- Request đến domain trên cổng `80` được Nginx nhận.
- Nginx chuyển request `/api/...` đến `127.0.0.1:8080` và các đường dẫn còn lại đến `127.0.0.1:3000`, đồng thời giữ các thông tin client cần thiết qua header.
- Người dùng truy cập domain mà không cần thêm `:3000` hoặc `:8080`.
- Khi upstream không sẵn sàng, Nginx thường báo `502`; log giúp xác định nguyên nhân.

## 9. Lỗi thường gặp và cách khắc phục

| Hiện tượng                                    | Kiểm tra/cách khắc phục                                                                                                                                     |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `502 Bad Gateway`                             | Kiểm tra upstream tương ứng với URL: frontend cổng `3000`, API cổng `8080`; xác nhận service đang chạy/bind đúng địa chỉ và xem `/var/log/nginx/error.log`. |
| `504 Gateway Timeout`                         | Kiểm tra ứng dụng có bị treo/chậm, tài nguyên VPS và log ứng dụng; chỉ tăng timeout khi đã xác định nhu cầu.                                                |
| `nginx -t` báo lỗi                            | Sửa lỗi cú pháp hoặc dấu `;`/`{}` ở file và dòng được báo; chưa reload cho đến khi kiểm tra thành công.                                                     |
| Domain không truy cập được                    | Kiểm tra DNS trỏ về VPS, firewall/security group cho phép cổng `80`, Nginx đang chạy và `listen 80` đúng.                                                   |
| Trả nhầm website/server block                 | Kiểm tra `server_name`, các cấu hình site được enable và server block trùng; kiểm tra bằng request có header `Host`.                                        |
| Nginx không kết nối được dù app chạy          | Kiểm tra dịch vụ tương ứng có thực sự nghe trên `127.0.0.1:3000` hoặc `127.0.0.1:8080` không; `localhost` có thể resolve sang IPv6.                         |
| Cổng 3000/8080 truy cập trực tiếp từ Internet | Rà soát địa chỉ bind và firewall; thông thường client chỉ cần cổng `80` (hoặc `443` khi triển khai HTTPS).                                                  |

## 10. Kết luận ngắn để trình bày

“Nginx Reverse Proxy nhận HTTP trên cổng 80 theo domain, định tuyến `/api/...` tới API Gateway nội bộ ở `127.0.0.1:8080` và các đường dẫn khác tới React ở `127.0.0.1:3000`. Các header proxy giúp upstream biết host, IP client và giao thức ban đầu. Khi upstream không phản hồi hợp lệ, lỗi thường gặp là `502 Bad Gateway`; còn `504` biểu thị quá thời gian chờ và `500` là lỗi nội bộ. Trước khi áp dụng cấu hình, cần kiểm tra và sao lưu file hiện có, chạy `nginx -t`, rồi mới reload.”

## Trạng thái thực hiện

- **Đã thực hiện trong workspace:** tạo tài liệu hướng dẫn `README.md`.
- **Chưa thực hiện:** truy cập VPS, đọc/sửa `/etc/nginx/sites-available/default`, cài/reload Nginx, kiểm tra domain hoặc dừng ứng dụng. Các việc này cần bạn tự thực hiện trên VPS theo hướng dẫn ở trên.
- **Mã nguồn React:** workspace không có mã nguồn để kiểm tra; không có mã ứng dụng nào bị sửa.

---

# Bài 3: Bảo mật HTTPS với Let's Encrypt và Certbot

> Đây là tài liệu học tập và cấu hình tham khảo. Workspace không kết nối tới VPS. Chưa có DNS, firewall, package, chứng chỉ hay dịch vụ Nginx nào được kiểm tra hoặc thay đổi trên máy chủ thực tế. Các lệnh có `sudo` có thể yêu cầu mật khẩu quản trị.

## 1. Mục tiêu kiến thức

- Kiểm tra DNS, Nginx và khả năng truy cập từ Internet trước khi xin chứng chỉ.
- Dùng Certbot và plugin Nginx để lấy chứng chỉ Let's Encrypt cho `api.my-ecommerce.com`.
- Phục vụ HTTPS trên cổng `443`, chuyển HTTP sang HTTPS và proxy tới API Gateway nội bộ.
- Kiểm tra cơ chế tự gia hạn và lưu kết quả kiểm thử thực tế, không suy đoán kết quả.

## 2. Mô hình HTTP và HTTPS

```text
Client -- HTTP :80 --> Nginx -- 301 --> https://api.my-ecommerce.com
Client -- HTTPS :443 (TLS) --> Nginx -- HTTP nội bộ --> 127.0.0.1:8080 (API Gateway)
```

HTTP cổng `80` chỉ chuyển hướng tới HTTPS; request HTTPS được Nginx giải mã TLS rồi proxy tới backend qua loopback. Kết nối từ Nginx tới `127.0.0.1:8080` trong cấu hình mẫu vẫn là HTTP nội bộ, không phải TLS end-to-end.

## 3. Kiểm tra điều kiện trước khi cấp chứng chỉ

Let's Encrypt phải xác minh quyền kiểm soát domain. Với HTTP-01 phổ biến, máy chủ xác minh cần truy cập được challenge qua HTTP cổng `80`. Hãy hoàn tất các kiểm tra sau trước khi chạy Certbot.

1. **Kiểm tra DNS A và AAAA:**

   ```bash
   dig +short A api.my-ecommerce.com
   dig +short AAAA api.my-ecommerce.com
   ```

   Bản ghi A phải trả về IPv4 public hiện tại của VPS. So sánh với địa chỉ được nhà cung cấp VPS cấp (địa chỉ `ip addr` có thể là private nếu VPS dùng NAT). Nếu có AAAA, IPv6 phải trỏ đúng máy chủ và máy chủ/firewall/Nginx phải phục vụ domain qua IPv6. AAAA sai có thể làm một số xác minh hoặc client đi nhầm máy; sửa DNS chỉ qua nhà cung cấp DNS sau khi bạn xác nhận.

2. **Kiểm tra phiên bản Ubuntu và Nginx:**

   ```bash
   cat /etc/os-release
   nginx -v
   sudo systemctl status nginx
   sudo nginx -t
   ```

   `nginx -t` cần báo cấu hình hợp lệ trước khi Certbot chỉnh hoặc reload Nginx. Đọc nội dung Nginx đang dùng trước khi thay đổi:

   ```bash
   sudo nginx -T
   ```

   Lệnh trên in cấu hình hợp nhất; không chia sẻ đầu ra công khai nếu cấu hình có thông tin nhạy cảm.

3. **Kiểm tra listener và firewall:**

   ```bash
   sudo ss -lntp
   sudo ufw status verbose
   ```

   Xác nhận Nginx có thể lắng nghe cổng `80` và `443`. Ngoài UFW, hãy kiểm tra firewall/security group tại trang quản trị nhà cung cấp VPS. Cả TCP `80` lẫn TCP `443` phải được cho phép từ Internet. Đây là bước kiểm tra, không tự mở firewall.

4. **Kiểm tra domain từ một máy bên ngoài VPS:**

   ```bash
   curl -I --max-time 10 http://api.my-ecommerce.com
   ```

   Phản hồi HTTP chứng tỏ request đã tới một web server; timeout/không phân giải được tên miền cần điều tra DNS, route hoặc firewall. Nên kiểm tra từ mạng ngoài VPS vì truy cập từ chính VPS không xác nhận được firewall inbound của nhà cung cấp. Chưa cần HTTPS hoạt động ở bước này.

Không thay đổi DNS, firewall, cấu hình Nginx hoặc dịch vụ hệ thống nếu chưa xem cấu hình hiện tại và chưa được bạn xác nhận.

## 4. Kiểm tra và cài Certbot trên Ubuntu

Trước hết xác định Ubuntu và cách Certbot đang được cài, tránh cài trùng:

```bash
cat /etc/os-release
command -v certbot
certbot --version
snap list certbot
dpkg-query -W -f='${Status} ${Version}\n' certbot python3-certbot-nginx
apt-cache policy certbot python3-certbot-nginx
```

Một số lệnh kiểm tra có thể báo package chưa cài; đó là thông tin, không phải lý do để cài cả hai phương thức. Với Ubuntu hiện đại, Certbot upstream thường khuyến nghị Snap; hãy xác nhận phương án phù hợp phiên bản Ubuntu/chính sách máy chủ trước khi cài. Chọn **một** trong hai phương án dưới đây, không duy trì đồng thời bản Snap và APT.

### Phương án A: Snap (thường được khuyến nghị)

Chỉ dùng nếu chưa có Certbot từ APT/Snap và Snap phù hợp với máy chủ:

```bash
sudo apt update
sudo apt install snapd
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
/snap/bin/certbot --version
```

Nếu `snap list certbot` đã cho thấy Certbot được cài, không cài lại. Nếu đang có bản APT, dừng lại và chọn cách chuyển đổi có chủ đích; không chạy cả hai bản.

### Phương án B: Ubuntu APT

Chọn nếu package APT của phiên bản Ubuntu và chính sách máy chủ là phù hợp, đồng thời chưa có bản Snap:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
certbot --version
```

APT cung cấp Certbot cùng plugin tích hợp Nginx. Không cài gói APT chồng lên Certbot Snap hoặc ngược lại. Không lệnh cài package nào trong tài liệu này đã được chạy.

## 5. Sao lưu và cấp chứng chỉ

Certbot với plugin Nginx có thể sửa file cấu hình và reload Nginx. Trước khi chạy:

1. Đọc cấu hình hiện tại bằng `sudo nginx -T`, xác nhận server block nhận `api.my-ecommerce.com` và `sudo nginx -t` thành công.
2. Sao lưu cấu hình Nginx hiện có. Ví dụ, sau khi xác nhận thư mục `/etc/nginx` tồn tại và đủ dung lượng:

   ```bash
   sudo cp -a /etc/nginx "/etc/nginx.backup.$(date +%Y%m%d-%H%M%S)"
   ```

3. Đảm bảo DNS đúng, port `80` truy cập được từ Internet, và bạn chủ động đồng ý để Certbot thay đổi cấu hình. Nếu chưa đủ điều kiện, chưa chạy lệnh cấp chứng chỉ.

Lệnh xin chứng chỉ bằng Nginx plugin và yêu cầu tự cấu hình HTTP redirect:

```bash
sudo certbot --nginx -d api.my-ecommerce.com --redirect
```

- `certbot`: chạy công cụ quản lý chứng chỉ.
- `--nginx`: dùng plugin Nginx để xác minh domain và cài cấu hình chứng chỉ.
- `-d api.my-ecommerce.com`: tên miền cần đưa vào chứng chỉ.
- `--redirect`: yêu cầu cấu hình chuyển hướng HTTP sang HTTPS.

Certbot có thể hỏi email, điều khoản sử dụng và xác nhận khác. Nếu chạy lệnh không có `--redirect`, chọn tùy chọn redirect khi Certbot hỏi nếu muốn chuyển HTTP sang HTTPS. Chỉ coi là thành công sau khi lệnh kết thúc thành công và kiểm tra được chứng chỉ/configuration.

Nếu muốn tự quản lý server block bằng tay thay vì để Certbot cài cấu hình, dùng `sudo certbot certonly --nginx -d api.my-ecommerce.com` để chỉ xin chứng chỉ. Sau khi chứng chỉ có thật, mới triển khai mẫu ở mục 6. Chọn cách này hoặc `certbot --nginx --redirect`; không bật đồng thời hai bộ server block trùng domain.

Kiểm tra sau khi Certbot cấp thành công:

```bash
sudo certbot certificates
sudo openssl x509 -in /etc/letsencrypt/live/api.my-ecommerce.com/fullchain.pem -noout -checkhost api.my-ecommerce.com -dates
```

Không tạo file chứng chỉ giả, không tạo/ghi đè private key do Certbot quản lý và không in hoặc commit nội dung private key.

## 6. Cấu hình Nginx HTTPS hoàn chỉnh

Mẫu cũng có trong [nginx/https-api.conf](nginx/https-api.conf). Đây là file tham khảo trong workspace, **không** phải cấu hình đã cài vào `/etc/nginx/sites-available/`. Chỉ dùng sau khi chứng chỉ thật đã tồn tại tại hai đường dẫn dưới đây. Chọn cấu hình Certbot tự quản lý hoặc cấu hình thủ công này; không kích hoạt server block trùng nhau.

```nginx
# HTTP chỉ chuyển hướng sang HTTPS; không proxy ở server block này.
server {
    listen 80;
    server_name api.my-ecommerce.com;

    return 301 https://api.my-ecommerce.com$request_uri;
}

# HTTPS kết thúc TLS tại Nginx, sau đó proxy tới backend nội bộ.
server {
    listen 443 ssl;
    server_name api.my-ecommerce.com;

    ssl_certificate /etc/letsencrypt/live/api.my-ecommerce.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.my-ecommerce.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

HTTP server block chỉ `return 301` sang hostname HTTPS cố định; HTTPS server block không redirect ngược về HTTP nên không tạo redirect loop. `fullchain.pem` chứa chứng chỉ máy chủ và chuỗi intermediate cần cho client xác minh. `privkey.pem` là private key nhạy cảm; bảo vệ quyền truy cập file, không gửi nội dung qua chat/email/Git. Nếu backend không phải API Gateway hoặc không nghe trên `127.0.0.1:8080`, thay đúng upstream/port sau khi xác minh. Nếu API chỉ phục vụ dưới prefix nhất định, điều chỉnh `location`/`proxy_pass` theo contract backend.

Nếu đã cấu hình AAAA và IPv6 hoạt động đầy đủ, cân nhắc thêm `listen [::]:80;` và `listen [::]:443 ssl;` tương ứng; không quảng bá AAAA tới máy không nhận được kết nối IPv6.

## 7. Tự động gia hạn chứng chỉ

Chứng chỉ Let's Encrypt thường có thời hạn khoảng 90 ngày. Certbot thường cài cơ chế gia hạn định kỳ; hãy kiểm tra đúng cơ chế của phương thức đã chọn, kiểm thử bằng dry-run và sửa lỗi ngay, không chờ tới ngày hết hạn. Không bật timer và cron cùng chạy renew.

### Phương án A: systemd timer

```bash
systemctl list-timers --all | grep -i certbot
sudo systemctl status certbot.timer
sudo systemctl status snap.certbot.renew.timer
```

Bản APT thường dùng `certbot.timer`; bản Snap thường dùng `snap.certbot.renew.timer`. Kiểm tra timer tương ứng với cài đặt thực tế; lệnh kiểm tra timer không tồn tại có thể báo lỗi bình thường nếu đó không phải phương thức đã cài. systemd timer thường được ưu tiên khi package đã cài và kích hoạt sẵn vì lịch chạy được quản lý bởi systemd, không phải tự tạo lịch cron thứ hai.

Xem lịch sử (dùng unit tương ứng):

```bash
sudo journalctl -u certbot.service --since '7 days ago'
sudo journalctl -u snap.certbot.renew.service --since '7 days ago'
sudo tail -n 100 /var/log/letsencrypt/letsencrypt.log
```

### Phương án B: cronjob (chỉ khi không có timer/job khác)

Chỉ tạo cron sau khi xác nhận không có systemd timer, cronjob hay lịch renew do package quản lý. Dùng `command -v certbot` để xác định đường dẫn binary. Ví dụ trong `/etc/cron.d/certbot` cho bản APT:

```cron
17 3,15 * * * root /usr/bin/certbot renew -q
```

`certbot renew` kiểm tra từng chứng chỉ và chỉ gia hạn khi gần tới thời điểm cần; chạy hai lần mỗi ngày không đồng nghĩa xin chứng chỉ mới mỗi lần. Không cài ví dụ này nếu timer đã hoạt động.

Nếu cần reload Nginx sau khi chứng chỉ được gia hạn, kiểm tra plugin/cấu hình hiện tại trước để tránh reload dư thừa. Có thể dùng deploy hook, hook chỉ được gọi sau lần gia hạn thành công. Ví dụ nội dung `/etc/letsencrypt/renewal-hooks/deploy/reload-nginx`:

```sh
#!/bin/sh
/usr/sbin/nginx -t && /bin/systemctl reload nginx
```

Tạo hook bằng editor có quyền quản trị và đặt quyền thực thi (`sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/reload-nginx`). Không cần thêm hook nếu Certbot/plugin hiện tại đã xử lý reload phù hợp. Sau đó kiểm thử:

```bash
sudo certbot renew --dry-run
```

Dry-run diễn tập quy trình gia hạn với staging, không thay thế chứng chỉ production; thành công nghĩa là renewal validation có thể hoàn tất. Nếu thất bại, đọc lỗi trên terminal và `/var/log/letsencrypt/letsencrypt.log`, kiểm tra DNS/port 80, timer và cấu hình Nginx. Deploy hook thường không chạy trong dry-run trừ khi bật `--run-deploy-hooks` trên phiên bản Certbot hỗ trợ.

## 8. Kiểm thử HTTPS và kết quả mong đợi

Sau khi Certbot đã cấu hình hoặc sau khi cài cấu hình thủ công, chạy theo thứ tự:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo certbot certificates
sudo certbot renew --dry-run
curl -I http://api.my-ecommerce.com
curl -I https://api.my-ecommerce.com
openssl s_client -connect api.my-ecommerce.com:443 -servername api.my-ecommerce.com -verify_return_error </dev/null
```

Chỉ reload nếu `nginx -t` thành công. Kết quả mong đợi:

- HTTP trả `301` và header `Location: https://api.my-ecommerce.com/...`.
- HTTPS hoàn tất TLS và chứng chỉ có SAN đúng `api.my-ecommerce.com`; `openssl s_client` kết thúc với `Verify return code: 0 (ok)` khi chain tin cậy và hostname hợp lệ.
- Request HTTPS được proxy đến API Gateway. Status phụ thuộc endpoint, method và trạng thái backend; không mặc định phải là `200`.
- `certbot renew --dry-run` báo diễn tập gia hạn thành công.

Nếu endpoint health đã biết, kiểm tra thêm bằng `curl -i https://api.my-ecommerce.com/<health-path-thuc-te>`; thay bằng path thực tế, không đoán endpoint.

Ghi nhận sau khi bạn chạy lệnh trên VPS; hiện chưa có kết quả thực tế để điền:

| Kiểm tra                               | Kết quả mong đợi               | Kết quả thực tế / thời gian |
| -------------------------------------- | ------------------------------ | --------------------------- |
| `curl -I http://api.my-ecommerce.com`  | `301` tới HTTPS                | Chưa chạy                   |
| `curl -I https://api.my-ecommerce.com` | TLS hợp lệ; status tùy backend | Chưa chạy                   |
| `openssl s_client ...`                 | `Verify return code: 0 (ok)`   | Chưa chạy                   |
| `sudo certbot renew --dry-run`         | Dry-run thành công             | Chưa chạy                   |

Log Nginx và Certbot:

```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo tail -n 100 /var/log/letsencrypt/letsencrypt.log
systemctl list-timers --all | grep -i certbot
```

Thoát theo dõi log bằng `Ctrl+C`. Với systemd, xem `journalctl` ở mục gia hạn; với Snap/APT hãy dùng unit tương ứng đã tìm thấy.

## 9. Lỗi thường gặp và cách xử lý

| Hiện tượng                          | Kiểm tra và hướng xử lý                                                                                                                               |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| DNS chưa trỏ đúng VPS               | So sánh A với IPv4 public do nhà cung cấp cấp; kiểm tra AAAA nếu có; đợi DNS cập nhật rồi kiểm tra lại.                                               |
| Port 80 bị chặn                     | Kiểm tra UFW và firewall/security group. HTTP-01 cần truy cập challenge qua port 80; chỉ mở sau khi bạn xác nhận thay đổi.                            |
| Port 443 chưa mở                    | Mở TCP 443 ở firewall VPS và firewall nhà cung cấp sau khi được cho phép; nếu không, client không thể kết nối HTTPS.                                  |
| Certbot không xác minh domain       | Kiểm tra DNS A/AAAA, truy cập Internet tới port 80, Nginx chọn đúng `server_name`, và log Certbot.                                                    |
| Nginx sai đường dẫn chứng chỉ       | Xác nhận chứng chỉ đã cấp và file `live/.../fullchain.pem`, `privkey.pem` tồn tại; không tự tạo file giả.                                             |
| Chứng chỉ không đúng domain         | Chạy `sudo certbot certificates` và `openssl x509 ... -checkhost`; cấp lại đúng `-d` sau khi sửa DNS/challenge.                                       |
| Private key không khớp chứng chỉ    | Đọc lỗi `nginx -t`/error log, xác nhận hai file cùng lineage Certbot; không ghi đè key hoặc dán nội dung key.                                         |
| Gia hạn thất bại                    | Chạy `sudo certbot renew --dry-run`, đọc terminal, journal và `/var/log/letsencrypt/letsencrypt.log`; kiểm tra timer và HTTP-01 reachability.         |
| HTTP redirect loop                  | Đảm bảo redirect chỉ ở server `listen 80`; HTTPS server không redirect về HTTP; kiểm tra proxy/load balancer có truyền đúng scheme nếu có thêm proxy. |
| Backend API Gateway không hoạt động | Kiểm tra `curl -i http://127.0.0.1:8080/<endpoint>` trên VPS, listener và log backend; Nginx thường ghi `502` khi không kết nối được upstream.        |

## 10. Kết luận trình bày

“Let's Encrypt xác minh domain trước khi cấp TLS certificate. Certbot với Nginx plugin có thể cài chứng chỉ và tạo chuyển hướng HTTP sang HTTPS, nhưng cần kiểm tra DNS, port 80/443 và cấu hình hiện tại trước. Nginx kết thúc TLS trên cổng 443 rồi proxy request tới API Gateway nội bộ; Certbot tự gia hạn cần được kiểm tra bằng timer phù hợp và `renew --dry-run`. Private key phải được bảo vệ và không đưa vào Git.”

## Trạng thái bài 3

- **Đã thực hiện trong workspace:** bổ sung hướng dẫn bài HTTPS vào README và thêm file cấu hình tham khảo.
- **Chưa thực hiện trên VPS:** chưa truy cập/kiểm tra DNS, firewall, phiên bản Ubuntu, Nginx hay Certbot; chưa cài package, xin chứng chỉ, sửa cấu hình hệ thống, reload dịch vụ hoặc kiểm thử endpoint.
- **Việc bạn cần tự làm:** kiểm tra các điều kiện ở mục 3, chọn đúng một phương thức cài Certbot, sao lưu và xác nhận trước khi chạy Certbot hoặc chỉnh Nginx, rồi ghi kết quả thực tế ở mục 8.

---

# Bài 4: Tối ưu hóa Nginx và reload cấu hình không gián đoạn

> Hướng dẫn và cấu hình dưới đây chỉ là tài liệu tham khảo trong workspace. Không có kết nối tới VPS; chưa kiểm tra hay sửa cấu hình production, chưa reload/restart Nginx và chưa gửi request upload.

## 1. Mục tiêu bài học

- Cho phép request upload tới 10 MB tại phạm vi phù hợp.
- Bật Gzip có chọn lọc cho nội dung văn bản đủ điều kiện.
- Kiểm tra cấu hình trước khi áp dụng và xác minh hành vi sau reload.
- Phân biệt `restart` với `reload`, ưu tiên thao tác ít gián đoạn trên production.

Mẫu các đoạn cấu hình nằm trong [nginx/optimization.conf](nginx/optimization.conf). Đây là file tham khảo **không được Nginx tự động nạp**; không chép đè `/etc/nginx/nginx.conf` hoặc `/etc/nginx/sites-available/default`.

## 2. Nguyên nhân HTTP 413 và giới hạn upload

`413 Request Entity Too Large` (còn được gọi là `Payload Too Large`) nghĩa là request body vượt giới hạn của một tầng xử lý request. Nếu `client_max_body_size` ở Nginx nhỏ hơn body upload khoảng 3 MB, Nginx có thể từ chối request trước khi chuyển đầy đủ tới backend.

Nginx không phải tầng duy nhất có giới hạn. API Gateway, framework/backend, middleware multipart, reverse proxy/CDN phía trước, và validation của ứng dụng cũng có thể giới hạn kích thước. Nếu tăng giới hạn Nginx mà vẫn nhận 413, kiểm tra log/status tại backend và điều chỉnh tầng đang từ chối theo chính sách ứng dụng. Không tăng giới hạn toàn hệ thống tùy tiện nếu chỉ endpoint Avatar cần upload.

## 3. Cấu hình `client_max_body_size`

Nginx cho phép chỉ thị này trong ngữ cảnh `http`, `server` và `location`. Nếu không đặt ở phạm vi cụ thể hơn, giá trị được kế thừa từ phạm vi cha; chỉ thị đặt tại `location` sẽ ghi đè giá trị kế thừa trong location đó.

### Phương án A: toàn bộ domain

Thêm vào `server { ... }` tương ứng với domain, nếu tất cả route trên domain cần giới hạn giống nhau:

```nginx
server {
   server_name domain.com;

   client_max_body_size 10m;

   location /api/ {
      proxy_pass http://127.0.0.1:8080;
      # Giữ các proxy header đang dùng tại đây.
   }

   location / {
      proxy_pass http://127.0.0.1:3000;
   }
}
```

### Phương án B: chỉ endpoint Avatar

Nếu chỉ endpoint upload cần giới hạn 10 MB, đặt chỉ thị trong `location` khớp chính xác endpoint thật. Ví dụ dưới đây giả định endpoint là `/api/users/avatar`; hãy thay bằng route thực tế:

```nginx
server {
   server_name domain.com;

   location = /api/users/avatar {
      client_max_body_size 10m;
      proxy_pass http://127.0.0.1:8080;

      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
   }

   location /api/ {
      proxy_pass http://127.0.0.1:8080;
      # Giữ các proxy header đang dùng tại đây.
   }
}
```

Exact-match location cho route Avatar được ưu tiên trước location tiền tố `/api/`; các route API khác vẫn dùng cấu hình chung. Nếu endpoint có nhiều URI con, cân nhắc prefix phù hợp thay vì `=`. Với yêu cầu chỉ giới hạn Avatar, phương án B là lựa chọn hẹp hơn; phương án A đơn giản hơn nếu mọi request của domain được phép có body tới 10 MB.

`10m` là giới hạn kích thước **toàn request body** theo đơn vị Nginx, không chỉ riêng bytes của ảnh. Với `multipart/form-data`, boundary và các field khác cũng chiếm dung lượng, vì vậy file sát ngưỡng 10 MB có thể làm toàn body vượt `10m`. Nếu nghiệp vụ cần chấp nhận file đủ 10 MB cộng overhead, hãy thống nhất một giới hạn có phần đệm với backend và kiểm thử, thay vì tùy tiện tăng giá trị.

## 4. Cấu hình Gzip

Kiểm tra `/etc/nginx/nginx.conf` và các file include trước. Chỉ bổ sung các chỉ thị còn thiếu vào khối `http { ... }` hiện có; không lồng thêm `http {}` và không thay toàn bộ file. Không khai báo trùng nếu cấu hình đã có Gzip:

```nginx
http {
   # Giữ nguyên các thiết lập và include hiện có.

   gzip on;
   gzip_vary on;
   gzip_proxied any;
   gzip_comp_level 5;
   gzip_min_length 1024;
   gzip_types
      text/plain
      text/css
      text/javascript
      application/javascript
      application/json
      application/xml
      text/xml
      image/svg+xml;

   # Giữ nguyên các thiết lập và include còn lại.
}
```

Đoạn `http {}` trên minh họa đúng ngữ cảnh. Khi chỉnh file hệ thống, chỉ chèn các chỉ thị Gzip vào khối `http` có sẵn, không dán thêm một khối `http` lồng bên trong.

- `gzip on;`: bật nén Gzip cho response phù hợp khi client yêu cầu hỗ trợ.
- `gzip_vary on;`: thêm `Vary: Accept-Encoding` để cache phân biệt response nén và không nén theo khả năng client.
- `gzip_proxied any;`: cho phép Gzip cả response gửi cho request được chuyển tiếp qua proxy, theo điều kiện cấu hình này.
- `gzip_comp_level 5;`: mức nén từ 1 đến 9; mức cao hơn thường tốn CPU hơn. Mức 5 là lựa chọn cân bằng ban đầu, cần đo trên tải thực tế.
- `gzip_min_length 1024;`: chỉ nén body từ khoảng 1024 byte trở lên; Nginx dựa vào `Content-Length` để áp dụng điều kiện này.
- `gzip_types`: MIME type bổ sung được nén, ở đây là CSS, JavaScript, JSON, XML và SVG. `text/html` đã được Nginx xử lý mặc định nên không cần liệt kê.

Gzip giảm số byte văn bản truyền qua mạng, không sửa hoặc giảm kích thước file gốc trên ổ đĩa. Không nên bật nén cho mọi định dạng: JPEG, PNG, WebP, ZIP, gzip và các file vốn đã nén thường không giảm đáng kể, có thể chỉ tốn CPU. Response chỉ được nén khi client gửi `Accept-Encoding: gzip`, MIME type/độ dài/status và cách upstream xử lý cho phép; response đã được nén sẵn thường không nên nén lần hai.

## 5. Quy trình triển khai an toàn trên production

Các lệnh dưới đây là hướng dẫn để bạn tự thực hiện sau khi có quyền và chấp thuận thay đổi. **Không** chạy `reload` hoặc sửa file production từ workspace này.

1. **Kiểm tra cấu hình và tìm đúng file đang có hiệu lực.**

   ```bash
   sudo nginx -T
   sudo grep -RInE 'client_max_body_size|gzip(_on|_vary|_proxied|_comp_level|_min_length|_types)?' /etc/nginx
   ```

   `nginx -T` cho biết file được include và server/location hiện có. Có thể cần xem thêm `sites-enabled` và cấu hình domain cụ thể. Không sửa một file chỉ vì tên của nó có vẻ đúng; xác định file thực sự được include và các directive đã khai báo.

2. **Sao lưu đúng file sẽ chỉnh.** Ví dụ, thay đường dẫn theo kết quả kiểm tra:

   ```bash
   sudo cp -a /etc/nginx/nginx.conf "/etc/nginx/nginx.conf.bak.$(date +%Y%m%d-%H%M%S)"
   sudo cp -a /etc/nginx/sites-available/default "/etc/nginx/sites-available/default.bak.$(date +%Y%m%d-%H%M%S)"
   ```

   Chỉ sao lưu file tồn tại và sẽ được chỉnh; nếu server block nằm ở file khác, sao lưu file đó thay cho `default`.

3. **Chỉnh sửa tối thiểu** bằng editor quản trị, ví dụ `sudoedit /etc/nginx/nginx.conf` cho Gzip và `sudoedit <file-server-block-thuc-te>` cho giới hạn upload. Giữ nguyên cấu hình không liên quan; thêm `client_max_body_size` ở server hoặc location được chọn. Không áp dụng file mẫu trong workspace bằng cách include toàn bộ.

4. **Kiểm tra cấu hình trước khi áp dụng:**

   ```bash
   sudo nginx -t
   ```

5. Nếu kiểm tra thất bại, **không reload/restart**. Đọc thông báo chỉ ra file/dòng, sửa lỗi rồi chạy lại `sudo nginx -t`.
6. Chỉ khi `nginx -t` báo cú pháp và kiểm tra thành công mới áp dụng:

   ```bash
   sudo systemctl reload nginx
   ```

7. Kiểm tra trạng thái và log sau khi reload:

   ```bash
   sudo systemctl status nginx
   sudo journalctl -u nginx --since "10 minutes ago"
   sudo tail -n 100 /var/log/nginx/error.log
   sudo tail -n 100 /var/log/nginx/access.log
   ```

8. Kiểm tra website, frontend, `/api/`, upload trong/ngoài giới hạn và Gzip như mục 8. Đối chiếu error/access log cùng log backend để xác nhận các route cũ vẫn hoạt động.

Kiểm tra trước giúp phát hiện lỗi cú pháp, directive sai ngữ cảnh, file chứng chỉ/include thiếu hoặc xung đột trước khi yêu cầu Nginx nạp cấu hình mới. `nginx -t` không chứng minh ứng dụng/backend hoạt động đúng; cần kiểm tra endpoint thực tế sau reload.

## 6. Restart và reload: lựa chọn trên production

**`sudo systemctl restart nginx`** dừng rồi khởi động lại service. Có thể có khoảng gián đoạn trong lúc chuyển đổi, tùy tình trạng Nginx, systemd, socket và cách triển khai. Đây không phải lựa chọn ưu tiên cho thay đổi cấu hình thông thường khi Nginx đang chạy ổn định.

**`sudo systemctl reload nginx`** yêu cầu Nginx nạp lại cấu hình. Với master process đang khỏe và cấu hình hợp lệ, Nginx có thể khởi chạy worker mới để nhận kết nối mới, rồi cho worker cũ hoàn tất kết nối đang xử lý trước khi thoát.

Đây là **graceful reload**: thay đổi cấu hình trong khi giảm tối đa ảnh hưởng đến kết nối hiện tại. Trong vận hành, người ta thường gọi là reload gần như không gián đoạn; không nên cam kết zero-downtime tuyệt đối. Kết quả còn phụ thuộc lỗi cấu hình/runtime, worker, kết nối dài hạn, module và ứng dụng upstream.

**Lệnh đầu tiên luôn là `sudo nginx -t`. Chỉ reload khi lệnh đó thành công.** Không dùng restart như cách thay thế cho kiểm tra cú pháp.

## 7. Kiểm thử sau reload

Chạy kiểm tra sau khi cấu hình đã được bạn áp dụng trên VPS:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx
sudo journalctl -u nginx --since "10 minutes ago"
sudo tail -n 100 /var/log/nginx/error.log
sudo tail -n 100 /var/log/nginx/access.log
```

Chỉ chạy reload nếu bước `nginx -t` thành công. Nếu vừa kiểm tra ở quy trình triển khai và chưa chỉnh thêm gì, vẫn có thể dùng status/log để xác nhận; không cần reload lần nữa nếu chưa có thay đổi mới.

### Xác minh Gzip bằng response thực tế

Ví dụ HEAD theo yêu cầu bài tập:

```bash
curl -sSI -H 'Accept-Encoding: gzip' https://domain.com/
```

HEAD không tải body và có thể không đi qua cùng logic như GET. Để kiểm tra response thực tế, dùng GET tới một file/text endpoint đủ lớn và đúng MIME type:

```bash
curl -sS -D - -o /dev/null -H 'Accept-Encoding: gzip' https://domain.com/path/to/large-text-resource
curl -sS -D - -o /dev/null -H 'Accept-Encoding: identity' https://domain.com/path/to/large-text-resource
```

Thay domain/path bằng endpoint thật. Khi điều kiện phù hợp, request đầu có `Content-Encoding: gzip`; với `gzip_vary on`, kiểm tra `Vary: Accept-Encoding`. Request `identity` so sánh trường hợp client không yêu cầu Gzip. Không phải mọi response đều có `Content-Encoding: gzip`: MIME type, body size, status, `Content-Length`, header từ upstream và cấu hình thực tế đều ảnh hưởng.

### Xác minh website, API và upload

```bash
curl -I https://domain.com/
curl -i https://domain.com/api/<health-endpoint-thuc-te>
curl -i -F 'avatar=@./avatar-3mb.jpg' https://domain.com/api/<upload-endpoint-thuc-te>
```

Thay các URL bằng domain/endpoint thực tế; thêm thông tin xác thực mà API yêu cầu. Dùng test account/file an toàn, tốt nhất trên staging. Xác nhận frontend, `/api/` và upload dưới giới hạn hoạt động. Upload vượt 10 MB cần bị từ chối theo chính sách, thường là 413; trong production chỉ chạy thử upload vượt ngưỡng khi được phê duyệt và có thể dọn dữ liệu thử.

Phân biệt tầng phát sinh lỗi bằng status/access log, Nginx error log và log API Gateway: `client intended to send too large body` trong error log thường chỉ Nginx từ chối body; nếu request tới backend và backend log/response báo 413 thì giới hạn nằm ở gateway/ứng dụng. Lỗi kết nối upstream như 502/504 khác với lỗi body size. Status 413 tự nó không cho biết chắc chắn tầng nào đã phát sinh.

## 8. Lỗi thường gặp và cách khắc phục

| Hiện tượng | Cách kiểm tra/xử lý |
|---|---|
| Vẫn nhận `413` sau khi đặt `10m` | Xác nhận request khớp đúng server/location, `nginx -T` cho thấy giá trị hiệu lực, đã reload thành công; sau đó kiểm tra giới hạn API Gateway/framework/multipart và body overhead. |
| Nginx báo `413` khi file nhỏ hơn 10 MB | Tổng multipart body có thể lớn hơn file; xác định tầng phát sinh qua error/access và backend log. |
| Gzip bật nhưng thiếu `Content-Encoding` | Dùng GET với nội dung text đủ lớn; kiểm tra `Accept-Encoding`, `Content-Type`, `Content-Length`, status, upstream compression và `gzip_types`. |
| Cấu hình lỗi sau chỉnh sửa | Không reload/restart; chạy `sudo nginx -t`, sửa đúng file/dòng được báo, chạy lại test. |
| Route frontend/API hỏng sau reload | Kiểm tra server/location, proxy headers, upstream 3000/8080, access/error log và health endpoint. |
| Reload không thành công | Xem `systemctl status`, `journalctl` và `nginx -t`; giữ Nginx đang chạy với cấu hình cũ, sửa lỗi rồi thử lại. |

## 9. Kết luận trình bày

“Để sửa lỗi 413, cần đặt `client_max_body_size 10m` tại phạm vi nhỏ nhất phù hợp, đồng thời kiểm tra giới hạn của API Gateway và ứng dụng. Gzip chỉ nén response văn bản phù hợp khi client hỗ trợ, không giảm kích thước file gốc. Trên production, cần sao lưu, chỉnh tối thiểu và chạy `sudo nginx -t` trước; nếu hợp lệ thì dùng `systemctl reload nginx` để Nginx graceful reload, thay vì restart cho thay đổi cấu hình thông thường. Sau đó kiểm tra route, upload, response Gzip và log.”

## Trạng thái bài 4

- **Đã thực hiện trong workspace:** thêm hướng dẫn bài 4 vào README và tạo cấu hình snippet minh họa.
- **Chưa thực hiện trên VPS:** chưa đọc/sửa cấu hình hệ thống, chưa thay đổi giới hạn body, bật Gzip, kiểm tra cấu hình bằng Nginx, reload/restart hoặc gửi request kiểm thử.
- **Việc bạn cần tự làm:** xác định file và endpoint upload thực tế, sao lưu file hệ thống liên quan, chọn phạm vi giới hạn, bổ sung directives Gzip không trùng, chạy `sudo nginx -t`; chỉ khi thành công mới reload và kiểm tra website/API/upload/Gzip cùng log.
