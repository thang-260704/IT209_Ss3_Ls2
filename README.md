# Bài 2: Cấu hình trang lỗi tùy chỉnh (Custom Error Page 404)

## Mục tiêu
- Nginx trả về trang **404 tự thiết kế** thay cho trang lỗi mặc định khi người dùng truy cập đường dẫn không tồn tại.
- Dùng chỉ thị `internal` để **chặn truy cập trực tiếp** vào `http://<IP>/404.html`.

## File nộp bài

| File | Vị trí trên máy chủ |
|---|---|
| [`my-web.conf`](my-web.conf) | `/etc/nginx/sites-available/my-web.conf` |
| [`404.html`](404.html) | `/var/www/my-web/html/404.html` |
| [`index.html`](index.html) | `/var/www/my-web/html/index.html` (trang chủ demo) |

## Môi trường

| Thông tin | Giá trị |
|---|---|
| Máy chủ | VPS AZVPS – Ubuntu 22.04.5 LTS |
| IPv4 | `160.187.229.88` |
| User thao tác | `devops` (dùng `sudo`), SSH qua cổng `2222` |
| Web root | `/var/www/my-web/html/` |

---

## Cấu hình Server Block

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/my-web/html;
    index index.html;

    error_page 404 /404.html;

    location = /404.html {
        root /var/www/my-web/html;
        internal;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Giải thích các chỉ thị chính:
- `error_page 404 /404.html;` có nghĩa là khi phát sinh lỗi 404, Nginx **chuyển hướng nội bộ** (internal redirect) sang URI `/404.html` để lấy nội dung trả về, đồng thời **giữ nguyên mã trạng thái 404**.
- `location = /404.html` là location khớp **chính xác** với URI `/404.html`.
- `internal;` quy định location này chỉ phục vụ các request nội bộ do Nginx tự sinh ra, như `error_page`. Request đến trực tiếp từ trình duyệt hoặc `curl` sẽ bị từ chối với mã **404**. Vì `error_page 404` vẫn có hiệu lực, người dùng vẫn nhìn thấy trang 404 tùy biến chứ không thấy trang lỗi mặc định.

## Các bước thực hiện

Đăng nhập vào máy chủ bằng `devops`:

```bat
ssh -i %USERPROFILE%\.ssh\id_ed25519 -p 2222 devops@160.187.229.88
```

### Bước 1 – Tạo web root, trang chủ và trang 404

```bash
sudo mkdir -p /var/www/my-web/html
sudo nano /var/www/my-web/html/index.html   # nội dung: index.html
sudo nano /var/www/my-web/html/404.html     # nội dung: 404.html
```

### Bước 2 – Tạo Server Block

```bash
sudo nano /etc/nginx/sites-available/my-web.conf   # nội dung: my-web.conf
```

### Bước 3 – Kích hoạt site mới

```bash
sudo ln -sf /etc/nginx/sites-available/my-web.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/ptit-web.conf
ls -l /etc/nginx/sites-enabled/
```

Cấu hình `ptit-web.conf` của session 02 được huỷ kích hoạt (file gốc vẫn còn trong `sites-available`), vì không thể có hai `default_server` cùng lắng nghe cổng 80.

### Bước 4 – Kiểm tra cú pháp và reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Kết quả thực tế:

```text
devops@wean2607:~$ ls -l /etc/nginx/sites-enabled/
total 0
lrwxrwxrwx 1 root root 38 Oct  2 07:47 my-web.conf -> /etc/nginx/sites-available/my-web.conf
devops@wean2607:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
devops@wean2607:~$ sudo systemctl reload nginx
```

![nginx -t](images/01-nginx-test.png)

---

## Kiểm tra (chạy trên máy cá nhân)

```text
C:\Users\ADMIN>curl -I http://160.187.229.88/invalid-path-demo
HTTP/1.1 404 Not Found
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 00:48:44 GMT
Content-Type: text/html
Content-Length: 1558
Connection: keep-alive
ETag: "6abefefc-616"

C:\Users\ADMIN>curl -I http://160.187.229.88/404.html
HTTP/1.1 404 Not Found
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 00:48:50 GMT
Content-Type: text/html
Content-Length: 1558
Connection: keep-alive
ETag: "6abefefc-616"
```

![curl kiểm tra](images/02-curl-test.png)

Phân tích kết quả:
- Cả hai request đều trả về **`HTTP/1.1 404 Not Found`**, đúng như yêu cầu.
- `Content-Length: 1558` và `ETag: "6abefefc-616"` giống nhau ở cả hai lần, vì `0x616 = 1558` byte. Điều này chứng tỏ Nginx trả về **chính file `404.html` tùy biến**, không phải trang lỗi mặc định của Nginx (trang mặc định chỉ khoảng 162 byte).
- Với `/404.html`, chỉ thị `internal` từ chối truy cập trực tiếp và sinh ra lỗi 404. Lỗi này lại được `error_page` chuyển nội bộ sang trang tùy biến, nên người dùng không bao giờ đọc được file này qua URL trực tiếp với mã 200.

Trình duyệt truy cập `http://160.187.229.88/abc-xyz` hiển thị trang 404 tùy biến:

![Trang 404 trên trình duyệt](images/03-browser-404.png)

## Đối chiếu yêu cầu

| Yêu cầu | Kết quả |
|---|---|
| Có `404.html` tùy biến trong `/var/www/my-web/html/` | ✅ |
| Server Block dùng `error_page 404 /404.html;` | ✅ |
| `location = /404.html { root ...; internal; }` | ✅ |
| `/invalid-path-demo` trả về 404 kèm trang tùy biến | ✅ `404 Not Found`, `Content-Length: 1558` |
| Truy cập trực tiếp `/404.html` trả về 404 | ✅ `404 Not Found` (bị chặn bởi `internal`) |
