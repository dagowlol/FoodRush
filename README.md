# FoodRush - Chạy bằng Docker

Hướng dẫn này giúp người khác clone code về là chạy được bằng Docker Compose,
chỉ cần tạo đúng một file `.env`.

> **Lưu ý quan trọng:** Repository này sử dụng **Git Submodules** cho backend (`FoodRush_BE`) và frontend (`FoodRush_FE`). Hãy làm theo hướng dẫn clone bên dưới để lấy đầy đủ code.

## 1. Yêu cầu

| Công cụ | Phiên bản | Ghi chú |
| --- | --- | --- |
| Docker Desktop | 26.0 trở lên | Hoặc Docker Engine + Docker Compose v2 |
| Docker Compose | v2 (có sẵn trong Docker Desktop) | Kiểm tra bằng `docker compose version` |

Không cần cài Java, Maven, Node.js hay MySQL trên máy.

## 2. Clone Repository (với Submodules)

### Cách 1: Clone một lần lấy hết (Khuyến nghị)
```bash
git clone --recurse-submodules https://github.com/<your-username>/FoodRush.git
cd FoodRush
```

### Cách 2: Đã clone rồi (quên `--recurse-submodules`)
```bash
git clone https://github.com/<your-username>/FoodRush.git
cd FoodRush
git submodule update --init --recursive
```

### Cách 3: Clone từng submodule thủ công (nếu cách trên lỗi)
```bash
git clone https://github.com/<your-username>/FoodRush.git
cd FoodRush
git submodule init
git submodule update --remote
```

> **Kiểm tra:** Sau khi clone xong, thư mục `FoodRush_BE/be/` và `FoodRush_FE/foodrush/` phải có code bên trong (không phải thư mục rỗng).

## 3. Chuẩn bị file `.env`

Tạo file `.env` ở thư mục gốc repo (cùng cấp với `docker-compose.yml`):

```bash
# macOS / Linux
cp .env.example .env

# Windows (PowerShell hoặc CMD)
copy .env.example .env
```

Sau đó mở `.env` và điền các giá trị cần thiết.

### Bắt buộc

| Biến | Mô tả |
| --- | --- |
| `DB_PASSWORD` | Mật khẩu user `foodrush` cho MySQL |
| `DB_ROOT_PASSWORD` | Mật khẩu root của MySQL, nên khác `DB_PASSWORD` |
| `JWT_SECRET` | Chuỗi bí mật ký JWT. Để trống thì mỗi lần restart đều phải đăng nhập lại |

Sinh `JWT_SECRET` bằng lệnh sau:

```bash
openssl rand -base64 48
```

Biến `SEED_ENABLED` mặc định là `true`, khi đó app tự tạo sẵn tài khoản
quản trị và nhân viên ở lần khởi động đầu tiên. Chi tiết ở mục 5.

### Nếu muốn đăng nhập bằng Google

Điền `GOOGLE_CLIENT_ID` và `GOOGLE_CLIENT_SECRET`, đồng thời khai báo trong
Google Cloud Console > APIs & Services > Credentials > Authorized redirect URIs:

```
http://localhost:8080/login/oauth2/code/google
```

> `8080` ở trên là giá trị `WEB_PORT` trong `.env`. Nếu bạn đổi cổng thì sửa cả URL này.

`OAUTH2_REDIRECT_URI` phải trỏ về trang xử lý callback của frontend, ví dụ
`http://localhost:8080/auth/callback`.

### Nếu muốn upload ảnh voucher

Điền `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`.
Bỏ trống thì chức năng upload sẽ báo lỗi, các chức năng khác vẫn chạy bình thường.

### Nếu muốn đăng ký bằng OTP (gửi email)

Điền `MAIL_USERNAME` và `MAIL_PASSWORD`. Với Gmail cần dùng **Mật khẩu ứng dụng**
(App Password), không phải mật khẩu tài khoản. Bật xác thực 2 bước trước rồi tạo
App Password tại <https://myaccount.google.com/apppasswords>.

## 4. Khởi động

```bash
docker compose up -d --build
```

Lần đầu build mất khoảng 5-10 phút (tải Maven dependencies và npm packages).

Theo dõi log:

```bash
docker compose logs -f backend
```

## 5. Truy cập

| Dịch vụ | Địa chỉ |
| --- | --- |
| Web | http://localhost:8080 |
| Swagger UI | http://localhost:8080/swagger-ui.html |
| Health check API | http://localhost:8080/actuator/health |

Trong `docker-compose.yml`, cổng `8080` của backend và `3306` của MySQL đang được
comment để tránh xung đột với môi trường đang chạy sẵn ở local. Nếu cần truy cập
trực tiếp (ví dụ để debug bằng MySQL client), bỏ comment dòng `ports` tương ứng
trong service `db` hoặc `backend`.

## 6. Tài khoản khởi tạo

Khi `SEED_ENABLED=true` (mặc định), lần khởi động đầu tiên app sẽ tự động:

1. Tạo schema bảng theo `JPA_DDL_AUTO`.
2. Tạo tài khoản **quản trị** theo `ADMIN_USERNAME` / `ADMIN_EMAIL`.
3. Tạo tài khoản **nhân viên** theo `SEED_STAFF_USERNAME` / `SEED_STAFF_EMAIL`.
4. Nâng lên vai trò `ADMIN` mọi tài khoản có email trùng `ADMIN_EMAIL`
   (áp dụng cả khi bạn đã đăng ký qua Google hoặc form đăng ký trước đó).

Tài khoản chỉ được tạo nếu username đó chưa tồn tại, nên khởi động lại nhiều lần
không sinh trùng dữ liệu. Tài khoản không dùng mật khẩu cố định sẽ được sinh
mật khẩu ngẫu nhiên 16 ký tự và ghi ra file trong Docker volume `staff-data`.

### Lấy mật khẩu đã sinh tự động

```bash
# Tài khoản quản trị
docker compose exec backend cat /app/data/admin-credentials.txt

# Tài khoản nhân viên
docker compose exec backend cat /app/data/staff-credentials.txt
```

File này nằm trong volume nên không bị mất khi restart container. Xem lại bất cứ lúc
nào cũng được, nhưng nên xoá đi sau khi đã đổi mật khẩu.

### Đặt mật khẩu cố định thay vì sinh tự động

Điền `ADMIN_PASSWORD` hoặc `SEED_STAFF_PASSWORD` trong `.env`. Khi đó không có file
credentials nào được sinh ra.

Lưu ý: mật khẩu đã tạo trước đó sẽ **không** bị đổi. Muốn đổi thì dùng nút **Sửa**
trong trang quản trị, hoặc xoá volume database rồi khởi động lại
(`docker compose down -v`, lưu ý mất toàn bộ dữ liệu).

### Đổi email của tài khoản quản trị

Nếu muốn tài khoản quản trị dùng email của bạn, hãy đặt:

```bash
ADMIN_EMAIL=your-email@gmail.com
```

Sau đó restart backend:

```bash
docker compose up -d --force-recreate backend
```

Hai hành vi tuỳ tình huống:

- Tài khoản `ADMIN_USERNAME` (mặc định `admin`) đã tồn tại: `ADMIN_EMAIL` chỉ có tác
  dụng nâng quyền tài khoản đang đăng nhập có email đó.
- Tài khoản `ADMIN_USERNAME` chưa tồn tại: nó sẽ được tạo mới với `ADMIN_EMAIL`
  làm email đăng nhập.

## 7. Các lệnh thường dùng

```bash
# Xem trạng thái
docker compose ps

# Khởi động lại
docker compose restart backend

# Dừng (giữ dữ liệu)
docker compose down

# Dừng và xoá luôn database
docker compose down -v

# Xem log riêng từng service
docker compose logs -f db
docker compose logs -f backend

# Rebuild sau khi sửa code
docker compose up -d --build backend
```

## 8. Cấu trúc

```
.
├── docker-compose.yml
├── .env.example
├── .env                  <- bạn tự tạo, không commit
├── FoodRush_BE/
│   └── be/
│       ├── Dockerfile
│       ├── .dockerignore
│       ├── pom.xml
│       └── src/
└── FoodRush_FE/
    └── foodrush/
        ├── Dockerfile
        ├── nginx.conf
        ├── .dockerignore
        ├── package.json
        └── src/
```

Luồng request sau khi đóng gói:

```
Trình duyệt → nginx (:8080) → /api/* , /oauth2/* , /login/*  →  backend (:8080 nội bộ) → MySQL
```

Frontend được build tĩnh (Vite) rồi phục vụ bằng nginx, nên không cần Node.js lúc chạy.

## 9. Chạy song song với bản local

Nếu bạn vẫn chạy `npm run dev` + `mvn spring-boot:run` ở local như khi dev,
cần chỉnh `CORS_ALLOWED_ORIGINS` trong `.env` để có cả hai origin:

```bash
CORS_ALLOWED_ORIGINS=http://localhost:8080,http://localhost:5173
```

## 10. Xử lý sự cố

**Backend báo `Communications link failure` hoặc thoát liên tục**
Backend chưa kết nối được MySQL. Kiểm tra `docker compose logs db` xem MySQL đã
healthy chưa; compose đã cấu hình `depends_on: service_healthy` nên thường chỉ cần
đợi. Nếu vẫn lỗi, kiểm tra `DB_PASSWORD` và `DB_ROOT_PASSWORD` trong `.env`.

**`exec: "java": executable file not found in $PATH`**
Image build bị lỗi. Chạy lại `docker compose build --no-cache backend`.

**Đổi secret trong `.env` nhưng app không nhận**
Compose chỉ nạp lại biến môi trường khi tạo lại container, không phải khi restart.
Dùng `docker compose up -d --force-recreate backend`.

**Quên mật khẩu admin vừa sinh tự động**
Xem lại file trong volume:

```bash
docker compose exec backend cat /app/data/admin-credentials.txt
```

Nếu file không còn, chỉ còn cách đặt `ADMIN_PASSWORD` rồi xoá tài khoản `admin`
trong trang quản trị (hoặc tạo tài khoản mới bằng nút **Tạo người dùng** và tự gán
vai trò Admin). Vì lần chạy sau app thấy username đã tồn tại nên sẽ không sinh lại.

**Tài khoản admin không có quyền quản trị**
Kiểm tra log backend có dòng `Granted ADMIN role...` không. Nếu không, `ADMIN_EMAIL`
đang trỏ tới email chưa đăng ký và tài khoản `ADMIN_USERNAME` đã tồn tại với vai
trò khác. Xem lại cả hai biến trong `.env`.

**Trang trắng, console báo lỗi CORS**
Kiểm tra `CORS_ALLOWED_ORIGINS` có đúng origin đang mở trên trình duyệt không,
bao gồm cả `http://` và cổng. Với bản Docker, web chạy ở `http://localhost:8080`.

**Nút đăng nhập Google báo lỗi redirect**
`OAUTH2_REDIRECT_URI` trong `.env` và Authorized redirect URI trong Google Cloud
Console phải khớp nhau, và phải dùng đúng `WEB_PORT` của bạn.

**Đổi `WEB_PORT` nhưng Google login hỏng**
Bắt buộc cập nhật cả hai URL liên quan (`localhost:${WEB_PORT}/login/oauth2/code/google`
trong Google Cloud Console và `OAUTH2_REDIRECT_URI` trong `.env`).

## 11. Làm việc với Submodules (Development)

Khi bạn sửa code trong `FoodRush_BE/` hoặc `FoodRush_FE/`, đây là quy trình:

### Commit thay đổi trong submodule
```bash
# 1. Vào thư mục submodule
cd FoodRush_BE/be
# hoặc
cd FoodRush_FE/foodrush

# 2. Commit như repo bình thường
git add .
git commit -m "Fix: mô tả thay đổi"
git push origin main   # push lên repo con (FoodRush_BE hoặc FoodRush_FE)

# 3. Quay về root repo, cập nhật pointer submodule
cd ../..
git add FoodRush_BE    # hoặc FoodRush_FE
git commit -m "chore: update FoodRush_BE submodule"
git push origin main
```

### Cập nhật submodule về commit mới nhất từ remote
```bash
# Cập nhật 1 submodule
cd FoodRush_BE/be
git pull origin main
cd ../..
git add FoodRush_BE
git commit -m "chore: update FoodRush_BE to latest"

# Hoặc cập nhật tất cả submodules cùng lúc
git submodule update --remote --merge
git add FoodRush_BE FoodRush_FE
git commit -m "chore: update all submodules"
```

### Checkout branch khác trong submodule
```bash
cd FoodRush_BE/be
git checkout develop   # hoặc branch-name
cd ../..
git add FoodRush_BE
git commit -m "chore: switch FoodRush_BE to develop branch"
```

> **Quan trọng:** Luôn push thay đổi lên repo con **trước**, sau đó mới cập nhật pointer ở root repo. Nếu làm ngược, người khác pull sẽ bị lỗi "commit not found".

## 12. Bảo mật

- File `.env` đã được thêm vào `.gitignore`, không commit lên Git.
- Chỉ commit `.env.example`.
- Trước khi nộp bài, hãy **thu hồi các secret đã commit vào lịch sử Git**:
  Google client secret, Cloudinary API secret, mật khẩu ứng dụng Gmail và
  `JWT_SECRET`. Nếu đã push lên remote công khai, coi như các secret đó bị lộ và
  phải tạo lại.