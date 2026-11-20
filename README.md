# DevOps Final – Đề 001: hướng dẫn Terminal IntelliJ

Dùng **Terminal IntelliJ → PowerShell**. Sau `ssh`, chính terminal đó chạy lệnh Linux trên server; `exit` để về Windows. Thay TEN_GITHUB, TEN_REPO, USER_SERVER, IP_SERVER bằng thông tin thật. GitHub Secrets, ảnh minh chứng và website cần thao tác giao diện.

**Đã kiểm tra local:** API trả đủ 8 phòng lab. Trang `/student.html` hiển thị thông tin sinh viên, danh sách phòng lab và APP_VERSION; `/api/info` cung cấp thông tin phiên bản dạng JSON. CI kiểm tra SHA thực tế từ ứng dụng. Điểm cuối cùng phụ thuộc rubric và minh chứng triển khai.

Kiến trúc: trình duyệt → Nginx (APP_PORT:80) → Spring Boot (8080) → MySQL (3306, không public).

## 1. Chuẩn bị

- Bật Docker Desktop (Linux containers); máy cần Git và OpenSSH.
- Server cần Git, curl, Docker Compose v2; tài khoản SSH riêng thuộc nhóm docker.
- Tạo repo GitHub **public**, tên `devops-final-de001-HOTENKHONGDAU`, không tạo README sẵn; nhánh main.
- Điền đủ student.env: tên, mã SV, lớp, email, GitHub username, port, project, tài khoản/mật khẩu MySQL.
- Giữ `EXAM_CODE="001"`, `MYSQL_DATABASE="ptit_lab_db"`. Project dạng `tenviettat-de001`, viết thường.
- APP_PORT từ 8080 đến 65535, không trùng người khác. Ví dụ dưới dùng **8089**; đổi port thì thay cả URL.
- Giữ Nginx `listen 80`, upstream `app:8080`; `/` redirect tới `/api/labs`.

Chạy tại thư mục chứa Dockerfile, student.env và docker-compose.yml:

```powershell
cd "C:\Users\TRONG TIN\Downloads\devops-final-app-de001\devops-final-app-de001"
git --version
docker version
docker compose version
```

## 2. Chạy Docker và xem JSON

```powershell
docker compose --env-file student.env config --quiet
docker compose --env-file student.env up -d --build --wait --wait-timeout 300
docker compose --env-file student.env ps
curl.exe -f --retry 12 --retry-delay 5 --retry-all-errors --max-time 10 http://localhost:8089/api/labs
Start-Process "http://localhost:8089/api/labs"
```

`config --quiet` không in gì và không báo lỗi là hợp lệ. `--wait` không thay thế kiểm tra HTTP khi app chưa có healthcheck. Phải thấy JSON danh sách 8 phòng lab.

## 3. Sửa lỗi Docker an toàn

Đọc log trước:

```powershell
docker compose --env-file student.env ps
docker compose --env-file student.env logs --tail 100 app nginx mysql
```

| Lỗi | Xử lý |
|---|---|
| Không kết nối Docker engine | Bật Docker Desktop, chờ sẵn sàng |
| Port already allocated | Chọn APP_PORT khác trong env rồi chạy lại up; không dừng dịch vụ người khác |
| 502 lúc mới khởi động | Chạy curl có retry ở phần 2; nếu vẫn lỗi, đọc log app |
| Access denied for user | Kiểm tra tài khoản/mật khẩu MySQL theo hướng dẫn bên dưới |
| Nginx lỗi sau khi thay container app | Chạy lệnh restart nginx bên dưới rồi kiểm tra API |

```powershell
docker compose --env-file student.env restart nginx
```

**Lỗi đã gặp:** `Access denied for user 'app_user' ... (using password: YES)` nghĩa là MySQL từ chối đăng nhập. App không khởi động được nên Nginx trả 502. Compose truyền MYSQL_PASSWORD trong env thành DB_PASSWORD của app. Có thể database cũ giữ mật khẩu khác; sửa env không tự đổi mật khẩu đã lưu. `using password: YES` chỉ nghĩa là có gửi mật khẩu.

**Chỉ nếu database local là dữ liệu mẫu được phép xóa**, tạo lại bằng:

```powershell
# CẢNH BÁO: -v xóa database của project hiện tại.
docker compose --env-file student.env down -v
docker compose --env-file student.env up -d --build --wait --wait-timeout 300
curl.exe -f --retry 12 --retry-delay 5 --retry-all-errors --max-time 10 http://localhost:8089/api/labs
```

MySQL tạo lại tài khoản từ env và chạy database/init.sql để thêm 8 phòng lab. Đây là cách đã sửa thành công trên máy local.

**Có dữ liệu cần giữ/server:** không dùng down -v. Sao lưu trước; khôi phục mật khẩu đúng trong env hoặc nhờ quản trị đổi mật khẩu đúng tài khoản MySQL, rồi tạo lại container app. Không đưa mật khẩu vào log/chat.

Dừng/chạy bình thường, giữ database:

```powershell
docker compose --env-file student.env down
docker compose --env-file student.env up -d --wait --wait-timeout 300
```

Tham khảo: [MySQL Docker](https://hub.docker.com/_/mysql), [Docker Compose up](https://docs.docker.com/reference/cli/docker/compose/up/).

## 4. Tạo SSH key cho pipeline

Chạy một lần; không ghi đè nếu đã có deploy_key. Khi hỏi passphrase, nhấn Enter hai lần:

```powershell
ssh-keygen -t ed25519 -f deploy_key
Get-Content .\deploy_key.pub
```

Copy toàn bộ dòng public key `ssh-ed25519 ...`. Đăng nhập server:

```powershell
ssh USER_SERVER@IP_SERVER
```

Nếu server yêu cầu key có sẵn, dùng thay cho lệnh trên:

```powershell
ssh -i "C:\duong-dan\khoa-server.pem" USER_SERVER@IP_SERVER
```

**Trên server Linux**, chạy:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat >> ~/.ssh/authorized_keys
```

Dán dòng public key → Enter → Ctrl+D. Tiếp tục:

```bash
chmod 600 ~/.ssh/authorized_keys
docker ps
docker compose version
git --version
curl --version
ss -ltn 'sport = :8089'
exit
```

docker ps phải chạy được không cần sudo. Thiếu quyền/công cụ thì nhờ quản trị server xử lý trước. Nếu port đã dùng, đổi APP_PORT; mở đúng cổng trên firewall/security group.

**Về PowerShell**, kiểm tra key mới:

```powershell
ssh -i .\deploy_key -p 22 root@221.121.5.186
```

Đăng nhập được thì gõ `exit`.

## 5. Thêm GitHub Secrets

Repo → Settings → Secrets and variables → Actions → New repository secret:

| Secret | Giá trị |
|---|---|
| SERVER_HOST | 221.121.5.186 |
| SERVER_USER | root |
| SERVER_SSH_KEY | Toàn bộ private key deploy_key, gồm BEGIN/END |

Copy key để dán trực tiếp vào secret:

```powershell
Get-Content -Raw .\deploy_key | Set-Clipboard
```

Không commit/nộp private key, token hoặc mật khẩu SSH. GITHUB_TOKEN được [GitHub tự cấp](https://docs.github.com/en/actions/concepts/security/github_token). Workflow hiện dùng SSH port 22, key không passphrase.

## 6. Commit và push

```powershell
git init
git branch -M main
git remote -v
```

Chưa có origin thì chạy:

```powershell
git remote add origin https://github.com/TEN_GITHUB/TEN_REPO.git
```

Có origin nhưng sai URL thì dùng thay cho lệnh trên:

```powershell
git remote set-url origin https://github.com/TEN_GITHUB/TEN_REPO.git
```

Kiểm tra trước khi commit:

```powershell
git check-ignore deploy_key
git add .
git diff --cached --name-only
git status --short
```

check-ignore phải in deploy_key. Danh sách staged không được có private key, .pem, token hoặc mật khẩu server. student.env phải được theo dõi vì workflow đọc file này; dùng mật khẩu DB riêng cho bài, không dùng lại mật khẩu thật ở nơi khác.

```powershell
git commit -m "Complete Docker and CI/CD configuration"
git push -u origin main
```

Nếu Git yêu cầu danh tính, đặt đúng thông tin rồi commit lại:

```powershell
git config user.name "HO TEN CUA BAN"
git config user.email "EMAIL_GITHUB_CUA_BAN"
```

Nếu cần đăng nhập GitHub, hoàn tất cửa sổ xác thực. Nếu push bị từ chối do remote có commit mới, không force push; kiểm tra lịch sử trước.

## 7. Kiểm tra CI/CD và server

GitHub → Actions → CI/CD: cả Build & Test và Deploy lên server phải xanh. Job đỏ: mở bước đỏ xem log để sửa. Pipeline tự build/test, push SHA/latest lên GHCR, pull image trên server và up --no-build; không cần tự push Docker.

```powershell
ssh -i .\deploy_key -p 22 root@221.121.5.186
```

Trên server:

```bash
cd ~/TEN_REPO
git rev-parse HEAD
docker compose --env-file student.env ps
curl -f --retry 12 --retry-delay 5 --retry-all-errors --max-time 10 http://127.0.0.1:8089/api/labs
exit
```

Về PowerShell:

```powershell
Start-Process "http://IP_SERVER:8089/"
```

Curl server được nhưng browser không vào: kiểm tra IP/firewall/security group. SSH lỗi: kiểm tra key và ba secrets. GHCR denied: kiểm tra bước login và quyền package của repo.

## 8. Minh chứng và nộp bài

Lưu ảnh thật vào screenshots/:

| File | Nội dung |
|---|---|
| 01-actions.png | Run trên main có cả hai job xanh |
| 02-ghcr.png | Package có tag commit SHA và latest |
| 03-compose-ps.png | Compose ps trên server, không phải local |
| 04-website.png | Browser hiển thị JSON từ IP server và port cá nhân |

Sau khi đủ ảnh, chạy ở local:

```powershell
New-Item -ItemType Directory -Force submission-content | Out-Null
Copy-Item screenshots\*.png submission-content\
git add README.md screenshots submission-content
git commit -m "Add submission guide and evidence"
git push origin main
```

Chờ pipeline cuối xanh trước hạn; đối chiếu ảnh với run/SHA được chụp. Nộp link repo public và thư mục bài lên RAIA; giữ website chạy tới khi có điểm. Không đưa private key vào gói nộp.

Nếu đề yêu cầu tải thư mục từ server bằng SFTP, vẫn dùng Terminal IntelliJ. Tạo thư mục nhận ngoài repository, thay đường dẫn key bằng đường dẫn thật:

```powershell
New-Item -ItemType Directory -Force ..\bai-nop | Out-Null
Set-Location ..\bai-nop
sftp -i "C:\DUONG_DAN_DAY_DU\deploy_key" USER_SERVER@IP_SERVER
```

Ở dấu nhắc sftp>, thay TEN_REPO rồi chạy:

```text
get -r TEN_REPO
bye
```

Server phải có commit minh chứng mới nhất trước khi tải. Sau đó dùng lệnh cd phần 1 để về dự án.

## Checklist trước khi nộp

- Repo public, đúng tên/main, student.env đủ và đúng quy ước.
- Dockerfile hai stage, Gradle + JDK 21 → JRE 21, non-root, APP_VERSION, EXPOSE, ENTRYPOINT exec. Dự án hiện dùng Gradle 8.14 với JDK 21.
- Compose context `.` vì Dockerfile cùng thư mục; DB qua biến, chỉ public Nginx, mount read-only.
- Hai job xanh, image SHA/latest, deploy không build trên server.
- Website server trả JSON, đủ bốn ảnh, submission-content và link RAIA trước hạn.
- Mở `/student.html` để chụp thông tin sinh viên, SHA và 8 phòng lab; đối chiếu SHA với run Actions. `/` vẫn chuyển tới `/api/labs`.

