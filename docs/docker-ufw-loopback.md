# Bài tập: Tích hợp UFW và Docker

> Workspace không có Docker Compose hay runtime VPS để kiểm tra. Nội dung và `docker-compose.yml` trong project là ví dụ theo đề bài; chưa có Docker, UFW hoặc Nginx nào được thay đổi.

## 1. Vì sao port Docker có thể lộ dù UFW chặn?

Khi Docker publish một container port lên host bằng cú pháp `8080:8080`, trong cấu hình bridge network thông thường Docker tạo quy tắc NAT/firewall để chuyển traffic tới container. Tùy phiên bản Docker, backend iptables/nftables và cấu hình UFW, traffic publish có thể được chuyển tiếp theo đường không đi qua các quy tắc UFW mà người dùng thường mong đợi. Vì vậy chỉ nhìn `ufw status` không đủ để kết luận port container không thể truy cập từ Internet.

Biện pháp phù hợp cho tình huống này là chỉ bind host port trên loopback: `127.0.0.1:8080:8080`. Nginx chạy trực tiếp trên cùng host vẫn gọi được địa chỉ loopback; remote clients không thể kết nối tới host port qua public interface trong thiết lập chuẩn. Đây là thu hẹp bề mặt truy cập, không phải bảo vệ tuyệt đối toàn máy chủ.

## 2. Compose đã sửa

File mẫu trong workspace là [docker-compose.yml](../docker-compose.yml):

```yaml
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "127.0.0.1:8080:8080"
```

Giải thích mapping:

- `8080:8080`: dạng `HOST_PORT:CONTAINER_PORT`. Không chỉ định host IP thường khiến Docker publish trên các giao diện host, thường là `0.0.0.0` (và có thể cả IPv6 tùy cấu hình). Khi đó máy ngoài có thể thử truy cập public-IP:8080.
- `127.0.0.1:8080:8080`: dạng `HOST_IP:HOST_PORT:CONTAINER_PORT`. Docker chỉ bind host port `8080` trên loopback IPv4. Port bên trong container vẫn là `8080`; image và tên service `backend` không đổi.

Nginx chạy trực tiếp trên host có thể tiếp tục cấu hình upstream như sau:

```nginx
proxy_pass http://127.0.0.1:8080;
```

Không cần mở host port `8080` ra public interface để Nginx trên cùng host truy cập backend.

**Khác biệt khi Nginx chạy trong container:** `127.0.0.1` bên trong container Nginx là chính container đó, không phải host hay backend container. Khi ấy nên đặt Nginx và backend vào cùng Docker network, proxy bằng service DNS name như `http://backend:8080`, và thường không cần publish backend port ra host. Không áp dụng máy móc cấu hình loopback host cho mô hình này.

## 3. Áp dụng có kiểm soát

Các lệnh dưới đây để bạn tự chạy sau khi xem Compose trên VPS và xác nhận thời điểm triển khai. `docker compose up -d` có thể tạo lại container khi mapping thay đổi; backend có thể gián đoạn ngắn trong lúc thay container. Không chạy lệnh này trong giờ cao điểm nếu chưa có kế hoạch.

1. Sao lưu/đọc file Compose đang triển khai; không ghi đè cấu hình khác chưa kiểm tra.
2. Xác thực cấu hình Compose trước:

   ```bash
   docker compose config
   ```

   Nếu hợp lệ, lệnh in cấu hình đã chuẩn hóa. Nó kiểm tra cú pháp/interpolation, không xác nhận port có thể truy cập hay ứng dụng khỏe.

3. Sau khi chấp thuận thay đổi và lên kế hoạch cho khả năng recreate container, áp dụng:

   ```bash
   docker compose up -d
   ```

4. Kiểm tra mapping container:

   ```bash
   docker compose ps
   docker compose port backend 8080
   ```

   Mapping mong đợi là `127.0.0.1:8080`, không phải `0.0.0.0:8080` hoặc `[::]:8080`.

5. Kiểm tra listener và backend từ host:

   ```bash
   sudo ss -lntp | grep ':8080'
   curl -i http://127.0.0.1:8080/
   docker compose logs backend
   ```

   `ss` có thể hiện `docker-proxy`, hoặc không hiện listener tường minh tùy Docker/network backend; hãy đối chiếu với `docker compose ps`/`docker compose port`. Endpoint `/` có thể trả `404` dù ứng dụng đã trả lời; thay bằng endpoint thật nếu có. `docker compose logs backend` giúp xem lỗi khởi động hoặc lỗi ứng dụng.

## 4. Xác nhận từ bên ngoài

Chạy kiểm tra từ một máy khác trên Internet, không phải từ VPS hoặc cùng loopback:

```bash
curl -i --connect-timeout 5 http://203.0.113.10:8080/
```

Thay `203.0.113.10` bằng public IPv4 thật của máy chủ. Kết quả mong đợi là không thiết lập được kết nối trực tiếp tới port `8080` (timeout hoặc connection refused), trong khi truy cập website/API qua Nginx vẫn hoạt động. Nếu port vẫn trả response, kiểm tra toàn bộ Compose project/container khác, listener khác trên host, port mapping thực tế, Docker firewall rules, public IPv6 và firewall/security group của nhà cung cấp.

UFW có thể tiếp tục được dùng như một lớp firewall bổ sung; nếu cần, xem trạng thái bằng `sudo ufw status verbose`. Không suy ra an toàn chỉ từ trạng thái UFW hoặc từ file Compose. Xác minh mapping, listener, IPv4/IPv6 và kết nối từ bên ngoài. Cũng nên dùng Docker Engine đã cập nhật; một số phiên bản cũ có hành vi đặc biệt với port publish trên localhost trong cùng L2 network.

## 5. Phạm vi bảo vệ

Bind loopback ngăn truy cập trực tiếp qua địa chỉ mạng công cộng trong cấu hình chuẩn, nhưng không bảo vệ toàn bộ máy chủ. Các tiến trình trên host vẫn có thể truy cập port đó; các port/container khác, quyền Docker socket, SSH, ứng dụng và cấu hình firewall vẫn cần được bảo vệ riêng. Nếu Nginx ở máy khác, container khác hoặc có proxy trung gian, thiết kế mạng/upstream cần thay đổi thay vì mở port backend công khai theo mặc định.

## Checklist

- [ ] Compose giữ nguyên image, service `backend` và container port `8080`.
- [ ] Host mapping chỉ là `127.0.0.1:8080`.
- [ ] `docker compose config` hợp lệ trước khi áp dụng.
- [ ] Đã tính tới khả năng `docker compose up -d` recreate container/gián đoạn ngắn.
- [ ] Nginx trên host gọi được `http://127.0.0.1:8080`.
- [ ] Một máy bên ngoài không truy cập trực tiếp được port `8080`.
- [ ] Đã kiểm tra các port/interface khác và không coi UFW là lớp bảo vệ duy nhất.
