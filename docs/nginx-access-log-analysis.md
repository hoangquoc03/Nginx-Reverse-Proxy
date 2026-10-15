# Bài tập: Phân tích access log Nginx nâng cao

## Pipeline Top 5 IP có nhiều HTTP 404 nhất

```bash
grep '" 404 ' /var/log/nginx/access.log \
  | awk '$9 == 404 { print $1 }' \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n 5
```

Pipeline chỉ đọc `/var/log/nginx/access.log`; không thay đổi hoặc xóa log.

## Ý nghĩa từng lệnh

1. `grep '" 404 '`: lọc các dòng có mẫu status nằm sau dấu nháy đóng của request và trước trường tiếp theo. Không dùng `grep 404` đơn giản vì chuỗi `404` có thể xuất hiện ở URL, User-Agent hoặc trường khác.
2. `awk '$9 == 404 { print $1 }'`: xác nhận trường status đúng bằng `404`, sau đó in trường đầu tiên là IP theo định dạng log mẫu.
3. `sort`: gom các IP trùng nhau cạnh nhau để `uniq` có thể đếm chính xác.
4. `uniq -c`: đếm số dòng liên tiếp giống nhau và in số lần xuất hiện cùng IP.
5. `sort -nr`: sắp xếp theo trường số ở đầu dòng (`-n`) theo thứ tự giảm dần (`-r`).
6. `head -n 5`: chỉ hiển thị tối đa năm dòng đầu.

## Kết quả với bốn dòng mẫu

Ba response có status `404`: `192.168.1.15` xuất hiện hai lần, `10.0.0.5` một lần. Dòng của `172.16.0.2` trả `200` nên bị loại. Kết quả mong đợi:

```text
      2 192.168.1.15
      1 10.0.0.5
```

Chỉ có hai IP phù hợp, nên Top 5 hiển thị hai dòng. Trường dung lượng `153` không được đếm vì `awk` so sánh `$9` với `404` và chỉ in `$1`.

### Giả định về định dạng log

Trong log Nginx `combined` phổ biến, IP là `$1`, request nằm trong dấu nháy kép và status là `$9` (bytes sent thường là `$10`). `grep` lọc theo vị trí mẫu thông thường, còn phép kiểm tra chính xác là `$9 == 404`. Nếu Nginx dùng `log_format` tùy chỉnh, vị trí trường có thể khác; kiểm tra định nghĩa `log_format` và điều chỉnh `$9`/`$1` cho đúng trước khi tin kết quả.

Nếu không có quyền đọc log, yêu cầu quyền phù hợp từ quản trị viên. Khi được phép, có thể dùng `sudo` chỉ cho bước đọc file:

```bash
sudo grep '" 404 ' /var/log/nginx/access.log \
  | awk '$9 == 404 { print $1 }' \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n 5
```

Không đổi quyền log thành world-readable hoặc dùng `chmod 777`. Trên Ubuntu, access log thường chỉ đọc được bởi `root` hoặc nhóm quản trị log; quyền cụ thể tùy hệ thống.

## Phương án chỉ dùng AWK

Có thể lọc, đếm và lấy Top 5 chỉ bằng AWK; đoạn dưới dùng vòng lặp chọn giá trị lớn nhất lặp tối đa năm lần:

```bash
awk '
$9 == 404 { count[$1]++ }
END {
    for (rank = 1; rank <= 5; rank++) {
        best = ""
        best_count = 0

        for (ip in count) {
            if (count[ip] > best_count ||
                (count[ip] == best_count && count[ip] > 0 && (best == "" || ip < best))) {
                best = ip
                best_count = count[ip]
            }
        }

        if (best == "") {
            break
        }

        printf "%d %s\n", best_count, best
        delete count[best]
    }
}
' /var/log/nginx/access.log
```

Cách này không cần `grep`, `sort`, `uniq` hoặc `head`; nó vẫn giả định IP ở `$1` và status ở `$9`. Nếu có nhiều IP, vòng quét tối đa năm lượt qua bảng đếm; cách pipeline thường dễ đọc hơn cho xử lý log thông thường.

## Diễn giải an toàn

Nhiều request `404` có thể cho thấy bot dò các đường dẫn phổ biến như `/.env`, nhưng số lượng 404 riêng lẻ **không đủ để kết luận IP độc hại**. Có thể là URL sai, crawler, client lỗi hoặc kiểm thử hợp lệ. Điều tra thêm tần suất theo thời gian, path/method, User-Agent, các status khác, log ứng dụng và bối cảnh nguồn truy cập trước khi chặn.

Nếu Nginx đứng sau CDN/reverse proxy, trường IP đầu tiên có thể là địa chỉ proxy, tùy `log_format` và Real IP configuration. Chỉ tin forwarded IP từ proxy đáng tin cậy; không lấy `X-Forwarded-For` tùy ý từ client làm căn cứ chặn.
