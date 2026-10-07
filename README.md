# Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Mục Tiêu
- Cấu hình máy chủ ảo Nginx (Server Block / Virtual Host) đóng vai trò làm **Reverse Proxy** định tuyến lưu lượng mạng.
- Áp dụng kỹ thuật **Path Matching** để phân chia luồng truy cập:
  - Phục vụ trực tiếp tài nguyên web tĩnh (Static Content) với hiệu năng cao.
  - Chuyển tiếp (Reverse Proxy) an toàn các yêu cầu động bắt đầu bằng tiền tố `/api/` đến ứng dụng backend Spring Boot chạy ở cổng 8082.

---

## 2. Bối Cảnh & Ràng Buộc
- **Bối cảnh:**
  - Hệ thống cần phục vụ người dùng truy cập trang chủ tĩnh tại địa chỉ IP của máy chủ qua cổng 80 mặc định.
  - Mọi yêu cầu API bắt đầu bằng tiền tố `/api/` phải được chuyển tiếp an toàn đến ứng dụng Spring Boot đang chạy tại cổng 8082 phía sau (`http://127.0.0.1:8082/`).
- **Ràng buộc triển khai:**
  - Tạo cấu trúc thư mục chứa trang tĩnh tại `/var/www/html/` và viết tệp `index.html` hiển thị thông tin học viên (Họ tên, Mã học viên, Mã lớp).
  - Tạo tệp cấu hình máy chủ ảo Nginx mới tại `/etc/nginx/sites-available/spring-proxy.conf`.
  - Cấu hình định tuyến:
    - Đường dẫn `/` (Static Serve): Trỏ vào thư mục `/var/www/html/`.
    - Đường dẫn `/api/` (Reverse Proxy): Chuyển tiếp (`proxy_pass`) đến ứng dụng Spring Boot tại `http://127.0.0.1:8082/`.
    - Thiết lập đầy đủ các Header HTTP chuyển tiếp (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) để Backend nhận diện chính xác danh tính Client.
  - Kích hoạt cấu hình bằng cách tạo liên kết mềm (symlink) sang thư mục `/etc/nginx/sites-enabled/`.
  - Kiểm tra cú pháp Nginx (`sudo nginx -t`) thành công trước khi reload dịch vụ.

---

## 3. Các Bước Thực Hiện & Lệnh Triển Khai

### Bước 1: Khởi tạo trang tĩnh và cấu trúc thư mục `/var/www/html/`
Tạo tệp `index.html` chứa thông tin học viên trong thư mục `/var/www/html/`:
```bash
sudo mkdir -p /var/www/html
sudo cp index.html /var/www/html/index.html
sudo chown -R www-data:www-data /var/www/html
sudo chmod 755 /var/www/html
sudo chmod 644 /var/www/html/index.html
```

### Bước 2: Thiết lập cấu hình Nginx Server Block
Tạo tệp cấu hình máy chủ ảo tại `/etc/nginx/sites-available/spring-proxy.conf`:
```bash
sudo nano /etc/nginx/sites-available/spring-proxy.conf
```

### Bước 3: Kích hoạt Virtual Host và hủy kích hoạt Site mặc định
Tạo liên kết mềm (symlink) sang thư mục `sites-enabled` và gỡ bỏ cấu hình mặc định `default` để tránh xung đột cổng 80:
```bash
# Gỡ bỏ liên kết cấu hình mặc định (nếu có)
sudo rm -f /etc/nginx/sites-enabled/default

# Kích hoạt cấu hình spring-proxy.conf
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
```

Kiểm tra liên kết vừa tạo:
```bash
ls -l /etc/nginx/sites-enabled/
```

### Bước 4: Kiểm tra cú pháp Nginx và Tải lại dịch vụ
Luôn kiểm tra cú pháp cấu hình trước khi áp dụng để đảm bảo hệ thống không bị gián đoạn:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. Tệp Cấu Hình Nginx (`spring-proxy.conf`)

Tệp cấu hình lưu trữ tại `/etc/nginx/sites-available/spring-proxy.conf`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name localhost;

    # 1. Static Serve: Định tuyến lưu lượng tĩnh / trỏ vào /var/www/html/
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # 2. Reverse Proxy: Chuyển tiếp toàn bộ yêu cầu có tiền tố /api/ đến Spring Boot Backend
    location /api/ {
        proxy_pass http://127.0.0.1:8082/;

        # Chuyển tiếp các HTTP Headers định danh Client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 5. Tệp Giao Diện Tĩnh (`index.html`)

Tệp hiển thị thông tin học viên lưu tại `/var/www/html/index.html`:

```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DevOps Fundamentals - Nginx Reverse Proxy</title>
    <style>
        :root {
            --bg-color: #0f172a;
            --card-bg: #1e293b;
            --text-primary: #f8fafc;
            --text-secondary: #94a3b8;
            --accent-color: #38bdf8;
            --success-color: #22c55e;
            --border-color: #334155;
        }
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-primary);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 20px;
        }
        .card {
            background-color: var(--card-bg);
            border: 1px solid var(--border-color);
            border-radius: 16px;
            padding: 40px;
            max-width: 540px;
            width: 100%;
            box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.5), 0 8px 10px -6px rgba(0, 0, 0, 0.4);
        }
        .badge {
            display: inline-flex;
            align-items: center;
            gap: 6px;
            padding: 6px 12px;
            border-radius: 9999px;
            font-size: 0.875rem;
            font-weight: 500;
            background-color: rgba(34, 197, 94, 0.15);
            color: var(--success-color);
            margin-bottom: 24px;
        }
        .dot {
            width: 8px;
            height: 8px;
            background-color: var(--success-color);
            border-radius: 50%;
            animation: pulse 2s infinite;
        }
        @keyframes pulse {
            0%, 100% { opacity: 1; transform: scale(1); }
            50% { opacity: 0.5; transform: scale(0.85); }
        }
        h1 {
            font-size: 1.75rem;
            font-weight: 700;
            margin-bottom: 12px;
            color: var(--text-primary);
        }
        .subtitle {
            color: var(--text-secondary);
            font-size: 1rem;
            margin-bottom: 28px;
            line-height: 1.5;
        }
        .info-list {
            background-color: rgba(15, 23, 42, 0.6);
            border: 1px solid var(--border-color);
            border-radius: 12px;
            padding: 20px;
            margin-bottom: 28px;
        }
        .info-item {
            display: flex;
            justify-content: space-between;
            padding: 10px 0;
            border-bottom: 1px solid rgba(51, 65, 85, 0.6);
        }
        .info-item:last-child {
            border-bottom: none;
        }
        .info-label {
            color: var(--text-secondary);
            font-size: 0.95rem;
        }
        .info-value {
            font-weight: 600;
            color: var(--accent-color);
            font-size: 0.95rem;
        }
        .footer {
            text-align: center;
            color: var(--text-secondary);
            font-size: 0.85rem;
            border-top: 1px solid var(--border-color);
            padding-top: 20px;
        }
    </style>
</head>
<body>
    <div class="card">
        <div class="badge">
            <span class="dot"></span> Nginx Static Web Server Active
        </div>
        <h1>Thông Tin Học Viên</h1>
        <p class="subtitle">Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot</p>
        
        <div class="info-list">
            <div class="info-item">
                <span class="info-label">Họ và tên:</span>
                <span class="info-value">Phan Thuận Thành</span>
            </div>
            <div class="info-item">
                <span class="info-label">Mã học viên:</span>
                <span class="info-value">176</span>
            </div>
            <div class="info-item">
                <span class="info-label">Mã lớp:</span>
                <span class="info-value">IT_209</span>
            </div>
            <div class="info-item">
                <span class="info-label">Môn học:</span>
                <span class="info-value">DevOps Fundamentals</span>
            </div>
            <div class="info-item">
                <span class="info-label">Định tuyến tĩnh:</span>
                <span class="info-value">/ -&gt; /var/www/html/</span>
            </div>
            <div class="info-item">
                <span class="info-label">Reverse Proxy:</span>
                <span class="info-value">/api/ -&gt; :8082</span>
            </div>
        </div>

        <div class="footer">
            Hệ thống Nginx Server Block Reverse Proxy &amp; Static Content Delivery
        </div>
    </div>
</body>
</html>
```

---

## 6. Kiểm Tra Kết Quả Thực Tế

### 6.1. Kiểm tra cú pháp Nginx
Lệnh kiểm tra:
```bash
sudo nginx -t
```

Đầu ra thực tế:
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### 6.2. Kiểm tra truy vấn HTTP tới Trang chủ tĩnh (`/`)
Lệnh kiểm tra Header HTTP:
```bash
curl -I http://localhost/
```

Đầu ra thực tế:
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 07 Oct 2026 08:35:32 GMT
Content-Type: text/html
Content-Length: 4925
Last-Modified: Wed, 07 Oct 2026 08:34:46 GMT
Connection: keep-alive
ETag: "6ac60426-133d"
Accept-Ranges: bytes
```

Lệnh kiểm tra nội dung phản hồi:
```bash
curl -s http://localhost/ | grep -E "Phan Thuận Thành|176|IT_209"
```

Đầu ra thực tế:
```html
                <span class="info-value">Phan Thuận Thành</span>
                <span class="info-value">176</span>
                <span class="info-value">IT_209</span>
```

---

### 6.3. Kiểm tra Reverse Proxy chuyển tiếp tới Backend Spring Boot (`/api/health`)
Lệnh kiểm tra Header HTTP:
```bash
curl -I http://localhost/api/health
```

Đầu ra thực tế:
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 07 Oct 2026 08:35:54 GMT
Connection: keep-alive
```

Lệnh kiểm tra nội dung phản hồi từ backend:
```bash
curl -i http://localhost/api/health
```

Đầu ra thực tế:
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 07 Oct 2026 08:35:46 GMT
Content-Length: 36
Connection: keep-alive

Spring Boot Application on port 8082
```

> **Ghi chú:** Kết quả trả về `HTTP/1.1 200 OK` cùng nội dung phản hồi `Spring Boot Application on port 8082`, chứng minh Nginx đã chuyển tiếp chính xác yêu cầu tới ứng dụng Spring Boot chạy trên cổng 8082 mà không xảy ra lỗi `502 Bad Gateway`.

---

### 6.4. Kiểm tra nhật ký truy cập (Access Log) của Nginx
Lệnh kiểm tra:
```bash
sudo tail -n 6 /var/log/nginx/access.log
```

Đầu ra thực tế:
```text
::1 - - [07/Oct/2026:08:35:32 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.18.0"
::1 - - [07/Oct/2026:08:35:37 +0000] "GET / HTTP/1.1" 200 4925 "-" "curl/8.18.0"
::1 - - [07/Oct/2026:08:35:46 +0000] "GET /api/health HTTP/1.1" 200 36 "-" "curl/8.18.0"
::1 - - [07/Oct/2026:08:35:54 +0000] "HEAD /api/health HTTP/1.1" 200 0 "-" "curl/8.18.0"
::1 - - [07/Oct/2026:08:36:01 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.21.0"
::1 - - [07/Oct/2026:08:36:08 +0000] "GET /api/health HTTP/1.1" 200 36 "-" "curl/8.21.0"
```

---

## 7. Đối Chiếu Kết Quả Mong Đợi

| Tiêu chí | Yêu cầu bài tập | Kết quả thực tế | Trạng thái |
| :--- | :--- | :--- | :---: |
| **Kiểm tra cú pháp Nginx** | `sudo nginx -t` trả về syntax is ok và test is successful | `syntax is ok` & `test is successful` | **ĐẠT** |
| **Trang chủ tĩnh (`/`)** | Phục vụ trực tiếp tại `/var/www/html/index.html`, trả về HTTP 200 | HTTP 200 OK, hiển thị đầy đủ Họ tên, Mã học viên, Mã lớp | **ĐẠT** |
| **Reverse Proxy (`/api/`)** | Chuyển tiếp tới backend Spring Boot cổng 8082 (`http://127.0.0.1:8082/`) | HTTP 200 OK, trả về nội dung từ Spring Boot backend | **ĐẠT** |
| **Tránh lỗi 502 Bad Gateway** | Không bị lỗi khi truy cập `/api/health` | Kết nối thành công, mã trạng thái HTTP 200 | **ĐẠT** |
| **Cấu hình Client Headers** | Thiết lập các trường header `Host`, `X-Real-IP`, `X-Forwarded-*` | Đã cấu hình đầy đủ trong khối `location /api/` | **ĐẠT** |
| **Kích hoạt Virtual Host** | Tạo liên kết mềm (symlink) sang thư mục `sites-enabled` | Đã liên kết và xác thực bằng `ls -l sites-enabled` | **ĐẠT** |

---

## 8. Kết Luận & Bài Học Kinh Nghiệm
1. **Mô hình Reverse Proxy:** Nginx đóng vai trò là một lớp trung gian an toàn giữa Client và Backend. Backend chỉ cần lắng nghe trên giao diện cục bộ (`127.0.0.1:8082`) mà không cần mở trực tiếp ra bên ngoài Internet, giúp tăng cường bảo mật hệ thống.
2. **Kỹ thuật Path Matching:** Nginx tối ưu hóa việc phân chia lưu lượng mạng. Với các tài nguyên tĩnh (`/`), Nginx đọc file và phục vụ trực tiếp với tốc độ cao; với các API động (`/api/`), Nginx chuyển tiếp tới ứng dụng Spring Boot.
3. **HTTP Header Forwarding:** Các chỉ thị `proxy_set_header` giúp bảo toàn thông tin về IP nguồn thực tế của người dùng và tên Host gốc, giúp backend ghi log kiểm toán và xử lý phân quyền chính xác.
