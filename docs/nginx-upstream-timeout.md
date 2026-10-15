# Bài tập: Tối ưu timeout cho Nginx Reverse Proxy

> Tài liệu hướng dẫn cho tình huống Spring Boot xử lý export khoảng 90 giây. Workspace không kết nối VPS; chưa có cấu hình hệ thống hay ứng dụng nào được kiểm tra hoặc thay đổi.

## 1. Vì sao có thể gặp `504 Gateway Time-out`?

Nginx nhận request rồi chờ upstream Spring Boot ở `localhost:8080`. Nếu thời gian chờ phản hồi upstream vượt timeout áp dụng cho route, Nginx thường kết thúc request và trả `504 Gateway Time-out`. Mặc định `proxy_read_timeout` thường là 60 giây; vì vậy một thao tác cần khoảng 90 giây có thể vượt ngưỡng này.

Đây là một nguyên nhân có khả năng, không phải kết luận duy nhất. Backend có thể đang chậm/kẹt, hoặc một load balancer, ingress, CDN, service mesh, client hay ứng dụng trung gian có deadline thấp hơn Nginx. Lỗi kết nối bị từ chối thường khác với upstream đã kết nối nhưng phản hồi quá chậm; hãy kiểm tra status và log ở từng tầng.

## 2. Cấu hình tham khảo

File mẫu trong workspace: [nginx/export-timeout.conf.example](../nginx/export-timeout.conf.example). Cấu hình dưới đây giữ timeout dài trong `location` của export thay vì tăng cho toàn domain:

```nginx
server {
    listen 80;
    server_name api.example.com;

    location /api/export/ {
        proxy_pass http://localhost:8080;

        # Backend trên cùng host nên thường phải kết nối nhanh.
        proxy_connect_timeout 5s;

        # Cho thời gian gửi request và nhận kết quả export dài.
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Thay hostname/upstream bằng giá trị thật. `proxy_pass` không có URI suffix nên request dưới `/api/export/` được chuyển tới backend với URI gốc. Nếu hệ thống có server block HTTPS, đặt location timeout tương ứng trong server block nhận HTTPS; không tạo server block trùng chỉ để thay timeout.

## 3. Ý nghĩa các timeout

- **`proxy_connect_timeout`**: thời gian tối đa Nginx chờ thiết lập kết nối tới upstream. Với backend local, `5s` là giá trị khởi đầu hợp lý; tăng thời gian xử lý báo cáo không cần kéo dài thời gian kết nối.
- **`proxy_send_timeout`**: thời gian tối đa giữa hai lần ghi liên tiếp khi Nginx gửi request tới upstream, không phải giới hạn tổng thời gian xử lý backend. `120s` cho phép việc gửi request chậm hơn; với export GET nhỏ, timeout này thường không phải nút thắt.
- **`proxy_read_timeout`**: thời gian tối đa giữa hai lần đọc liên tiếp từ upstream. `120s` cho phép backend mất khoảng 90 giây trước khi bắt đầu trả response, nếu trong khoảng đó không vượt quá 120 giây giữa các lần đọc.

**Điểm quan trọng:** `proxy_read_timeout 120s` không phải deadline tổng cho toàn bộ request. Nếu upstream liên tục gửi dữ liệu trong các khoảng ngắn hơn 120 giây, request có thể tiếp tục lâu hơn. Nếu cần giới hạn tổng thời lượng tuyệt đối, phải thiết kế deadline ở ứng dụng hoặc tầng điều phối phù hợp.

Với export khoảng 90 giây, `proxy_read_timeout 120s` tạo khoảng đệm khoảng 30 giây cho biến động thông thường. Đây không phải bảo đảm request luôn thành công; kiểm tra SLA, phân bố thời gian thực tế và timeout ở các hop khác. Giới hạn timeout hẹp cho route export để tránh giữ connection lâu cho toàn bộ API.

## 4. Kiểm tra và áp dụng an toàn

Các lệnh dưới đây là hướng dẫn để bạn tự chạy trên VPS. Không tự động sửa file hệ thống hoặc reload dịch vụ. Lệnh `sudo` có thể yêu cầu mật khẩu quản trị.

1. Xem cấu hình đang có và xác định file thực sự được include:

   ```bash
   sudo nginx -T
   ```

   Đọc đúng server/location hiện tại, tìm các timeout đã khai báo và tránh duplicate/conflicting location. Nếu dùng `sites-enabled` symlink, xác định file nguồn trước khi sửa.

2. Sao lưu file hiện có trước khi thay đổi, rồi chỉ chỉnh location cần thiết. Không chép đè toàn bộ `/etc/nginx/nginx.conf` bằng mẫu trong workspace.

3. Kiểm tra cấu hình:

   ```bash
   sudo nginx -t
   ```

   Chỉ tiếp tục nếu Nginx báo `syntax is ok` và `test is successful`. Nếu thất bại, không reload; sửa file/dòng được nêu và chạy lại test.

4. Chỉ sau khi test thành công mới reload:

   ```bash
   sudo systemctl reload nginx
   sudo systemctl status nginx
   ```

5. Thử export bằng đúng HTTP method, tham số và thông tin xác thực mà API yêu cầu. Ví dụ GET đơn giản:

   ```bash
   curl -i --max-time 130 http://api.example.com/api/export/
   ```

   `--max-time 130` là deadline của curl để client không tự ngắt trước Nginx; thay endpoint/method theo API thật. Sau đó xem Nginx error/access log và log Spring Boot để so sánh thời điểm request bắt đầu, timeout và thời điểm backend kết thúc.

## 5. Nếu vẫn nhận 504

- **Backend thật sự mất hơn 120 giây hoặc bị kẹt:** xem thời gian xử lý, thread pool, CPU/heap/GC, database query/connection pool, lock và dịch vụ phụ thuộc. Tăng timeout vô hạn chỉ giữ kết nối lâu hơn; cân nhắc export bất đồng bộ (trả job ID, client polling/tải file sau) nếu báo cáo có thời gian xử lý dài.
- **Timeout ở tầng giữa:** kiểm tra load balancer, ingress controller, CDN, service mesh, API gateway và proxy khác. Deadline thấp nhất trên đường đi sẽ có thể ngắt request trước Nginx.
- **Timeout phía client:** thư viện HTTP/browser/job runner có thể tự hủy request trước khi backend hoàn tất. Đối chiếu client timeout và log server.
- **Sai upstream hoặc kết nối không ổn định:** kiểm tra địa chỉ `localhost:8080`, trạng thái Spring Boot, bind IPv4/IPv6 và log `connect() failed`/upstream timeout. `localhost` đôi khi phân giải IPv6; nếu ứng dụng chỉ nghe IPv4, xác minh việc dùng `127.0.0.1` phù hợp.
- **Nginx vẫn báo upstream timeout:** đọc `sudo tail -n 100 /var/log/nginx/error.log` và `sudo journalctl -u nginx --since "10 minutes ago"`; đối chiếu log backend. Dòng kiểu `upstream timed out ... while reading response header from upstream` thường cho biết Nginx đang chờ response header từ upstream.

Thay đổi timeout Nginx không tự sửa giới hạn của Spring Boot hoặc các proxy khác. Mọi tầng cần có timeout nhất quán với thiết kế, và timeout cao hơn phải được cân nhắc về số lượng kết nối đồng thời/tài nguyên.

## Checklist

- [ ] Đã xác định đúng Nginx file và location cho `/api/export/`.
- [ ] `proxy_read_timeout` lớn hơn thời gian export dự kiến; các timeout khác đặt theo công việc tương ứng.
- [ ] Đã sao lưu và chỉ sửa cấu hình cần thiết.
- [ ] `sudo nginx -t` thành công trước khi reload.
- [ ] Đã kiểm tra log Nginx, backend và timeout của mọi tầng trung gian.
