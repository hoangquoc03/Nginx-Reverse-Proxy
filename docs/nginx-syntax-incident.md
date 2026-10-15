# Bài tập: Xử lý sự cố cú pháp Nginx

> Tình huống giả lập: Nginx báo lỗi khi đọc `/etc/nginx/sites-enabled/app.conf`. File dưới đây chỉ là ví dụ trong tài liệu; chưa có file hệ thống nào được mở hoặc sửa.

## 1. Nguyên nhân

Lỗi:

```text
nginx: [emerg] unexpected "proxy_set_header" in /etc/nginx/sites-enabled/app.conf:7
```

Nguyên nhân là dòng `proxy_pass` thiếu dấu chấm phẩy kết thúc directive:

```nginx
proxy_pass http://localhost:8080
```

Nginx cần dấu `;` để biết directive đã kết thúc. Khi parser gặp directive tiếp theo `proxy_set_header` trên dòng 7 mà chưa thấy dấu kết thúc, nó báo token đó là unexpected. Vì vậy vị trí được báo là dòng tiếp theo nơi parser không còn hiểu cú pháp, còn lỗi gốc nằm ngay cuối dòng `proxy_pass` phía trên.

## 2. File `app.conf` đã sửa

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Thay `example.com` bằng hostname thật và xác nhận backend thực sự lắng nghe tại địa chỉ/cổng được cấu hình. Đây là cấu hình mẫu, không được cài tự động vào `/etc/nginx/sites-enabled/app.conf`.

## 3. Ý nghĩa các directive

- `listen 80;`: nhận kết nối HTTP trên cổng TCP `80`.
- `server_name example.com;`: cho Nginx biết hostname mà virtual host này phục vụ. Chỉ thị này không tạo DNS record.
- `location / { ... }`: chọn xử lý cho các URI bắt đầu bằng `/`; trong ví dụ, các request khớp được proxy tới backend.
- `proxy_pass http://localhost:8080;`: chuyển request tới HTTP upstream tại `localhost:8080`. Dấu `;` ở cuối là bắt buộc. `localhost` phải phân giải tới địa chỉ mà backend đang bind; nếu backend chỉ nghe IPv4, có thể cần dùng `127.0.0.1`.
- `proxy_set_header Host $host;`: gửi hostname request ban đầu tới backend.
- `proxy_set_header X-Real-IP $remote_addr;`: gửi địa chỉ IP client mà Nginx quan sát được.

## 4. Kiểm tra cấu hình trước khi áp dụng

Trên VPS, trước tiên xem cấu hình đang có và xác định file nguồn:

```bash
sudo nginx -T
readlink -f /etc/nginx/sites-enabled/app.conf
```

Nếu `sites-enabled/app.conf` là symlink, thường nên chỉnh file tương ứng trong `sites-available` sau khi đã kiểm tra. Sao lưu file thực sự sẽ sửa rồi mới dùng editor quản trị, ví dụ `sudoedit <duong-dan-file-nguon>`; thay placeholder bằng đường dẫn đã xác minh.

Sau khi chỉnh, chạy:

```bash
sudo nginx -t
```

- **Thành công:** thường hiện `syntax is ok` và `test is successful`. Điều đó cho biết Nginx đọc được cấu hình và kiểm tra các tham chiếu cấu hình, không chứng minh backend tại `localhost:8080` đang chạy hay endpoint hoạt động.
- **Thất bại:** Nginx báo file và dòng liên quan. Không reload; sửa lỗi được chỉ ra rồi chạy lại `sudo nginx -t` cho tới khi thành công.

## 5. Reload sau khi kiểm tra thành công

Chỉ sau khi `sudo nginx -t` thành công mới áp dụng cấu hình:

```bash
sudo systemctl reload nginx
sudo systemctl status nginx
```

Reload yêu cầu Nginx nạp lại cấu hình đang chạy và thường phù hợp hơn restart cho thay đổi cấu hình thông thường. Không chạy reload khi test thất bại. Các lệnh `sudo` có thể yêu cầu mật khẩu quản trị; tài liệu này không chạy chúng thay bạn.

Để kiểm tra riêng backend, thực hiện request độc lập trên VPS:

```bash
curl -i http://127.0.0.1:8080/
```

Endpoint `/` có thể trả `404` dù server vẫn phản hồi; hãy thay bằng endpoint kiểm tra thực tế nếu có. Backend không hoạt động có thể gây lỗi proxy khi chạy ứng dụng, nhưng không nhất thiết làm `nginx -t` thất bại.

## 6. Nếu vẫn còn lỗi

Đọc journal của service và error log:

```bash
sudo journalctl -u nginx -n 50 --no-pager
sudo tail -n 50 /var/log/nginx/error.log
```

Sau đó kiểm tra các cấu hình được include, dấu `;` và cặp `{}` quanh server/location, các directive khai báo trùng, hostname/server block trùng, file symlink trong `sites-enabled`, và đường dẫn backend. `sudo nginx -T` hiển thị cấu hình sau khi include; đọc kỹ trước khi chia sẻ vì cấu hình môi trường có thể chứa thông tin nhạy cảm.

## Checklist

- [ ] Đã sửa dấu `;` sau `proxy_pass` trong đúng file nguồn.
- [ ] `sudo nginx -t` báo `syntax is ok` và `test is successful`.
- [ ] Chỉ sau khi test thành công mới reload Nginx.
- [ ] Đã kiểm tra trạng thái/journal sau reload.
- [ ] Đã kiểm tra backend `localhost:8080` riêng với kiểm tra cú pháp Nginx.
