# DevOps Đề 001 — làm nhanh trong 60 phút bằng Terminal IntelliJ

## 1. Thông tin đúng của bản đang làm

- Sinh viên: **Pham Ngoc Linh — B24DTCN360 — K24CNTT4**.
- Repo hiện tại: https://github.com/embedangyeunhathemattroi/devops (public, nhánh `main`).
- Server: `root@221.121.5.186`, thư mục `/root/devops`.
- Project Compose: `phamngoclinh-de001`; cổng **8089**.
- API: http://221.121.5.186:8089/api/labs
- Trang sinh viên/SHA: http://221.121.5.186:8089/student.html
- Actions: https://github.com/embedangyeunhathemattroi/devops/actions

Chỉ triển khai từ repo/thư mục này. Repo cũ `devops-final-de001-phamngoclinh` dùng cùng tên Compose project: chạy lại pipeline cũ có thể thay container của bài hiện tại, đổi cổng và gây lệch mật khẩu. Nếu đề bắt buộc tên repo `devops-final-de001-HOTENKHONGDAU`, phải thống nhất repo nộp với giảng viên; repo hiện tại vẫn tên `devops`.

**Nguyên nhân lỗi đã xử lý:** volume MySQL giữ tài khoản của lần chạy cũ; sửa `student.env` không đổi mật khẩu trong database. Đã sao lưu database trên server và đồng bộ riêng `app_user`, giữ dữ liệu 8 phòng lab. Mật khẩu quản trị cũ được giữ nguyên. Healthcheck mới thực sự đăng nhập và chạy `SELECT 1`, thay vì chỉ ping MySQL.

Không có quy trình bảo đảm tuyệt đối không lỗi hoặc chắc chắn 80 điểm. Các bước dưới giảm thao tác thừa và chỉ rõ nơi cần kiểm tra.

## 2. Chia 60 phút và kiểm tra trước (0–5 phút)

| Thời gian | Việc làm |
|---|---|
| 0–5 phút | Docker, thư mục, SSH, thông tin sinh viên |
| 5–15 phút | Build local, kiểm tra 8 phòng lab |
| 15–30 phút | Push, chờ hai job CI/CD |
| 30–40 phút | Kiểm tra server và truy cập từ máy cá nhân |
| 40–50 phút | Chụp 4 ảnh đúng repo/SHA/IP |
| 50–60 phút | Push minh chứng, kiểm tra run cuối, nộp |

Mở **IntelliJ → Terminal → PowerShell**. Chỉ copy phần lệnh trong khối, không copy dấu nhắc terminal. Sau `ssh`, terminal chuyển sang Linux; gõ `exit` để trở lại PowerShell.

```powershell
cd "C:\Users\TRONG TIN\Downloads\devops-final-app-de001\devops-final-app-de001"
git status --short
git remote -v
docker version
docker compose version
docker compose --env-file student.env config --quiet
ssh -o ConnectTimeout=10 -i .\deploy_key root@221.121.5.186 "docker compose version"
```

Docker lỗi kết nối: bật Docker Desktop, đợi engine sẵn sàng rồi chạy lại. `config --quiet` thành công không in gì. Không chạy `git init`, không tạo lại SSH key hoặc Secrets nếu bài đã cấu hình.

## 3. Chốt cấu hình một lần để tránh lỗi mật khẩu

Mở `student.env` trong IntelliJ, kiểm tra tên, mã SV, lớp, email, GitHub. Giữ:

```dotenv
GITHUB_USERNAME="embedangyeunhathemattroi"
EXAM_CODE="001"
APP_PORT="8089"
COMPOSE_PROJECT_NAME="phamngoclinh-de001"
MYSQL_DATABASE="ptit_lab_db"
MYSQL_USER="app_user"
```

- Giữ nguyên mật khẩu DB sau khi đã tạo volume. Nếu cần đổi, phải đổi trong MySQL và cấu hình cùng lúc; chỉ sửa env là chưa đủ.
- Luôn thêm `--env-file student.env` vào lệnh Compose.
- Không đổi tên project giữa chừng; tên project quyết định container/network/volume.
- Không chạy hai repo khác nhau có cùng project trên một server.
- Không chạy `down -v` trên server: lệnh đó xóa database. Khởi động lại bình thường không cần `down`.
- Chỉ dùng mật khẩu riêng cho bài; không dùng lại mật khẩu cá nhân trong file được commit.

## 4. Build và kiểm tra local (5–15 phút)

```powershell
$env:APP_VERSION = git rev-parse HEAD
docker compose --env-file student.env up -d --build --wait --wait-timeout 300
if ($LASTEXITCODE -ne 0) { throw "Docker chưa sẵn sàng: xem mục 7" }
docker compose --env-file student.env restart nginx
curl.exe -fsS --retry 12 --retry-delay 5 --retry-all-errors --max-time 10 http://localhost:8089/api/labs
$rooms = Invoke-RestMethod -TimeoutSec 10 http://localhost:8089/api/labs
if ($rooms.Count -ne 8) { throw "Phải có đúng 8 phòng lab" }
Invoke-RestMethod -TimeoutSec 10 http://localhost:8089/api/info
Start-Process "http://localhost:8089/student.html"
```

`APP_VERSION` ở local là HEAD tại thời điểm build. Nếu vừa sửa mã chưa commit thì chưa được dùng ảnh local để khẳng định mã trùng commit. Minh chứng nộp lấy từ server sau pipeline xanh.

## 5. Push và CI/CD (15–30 phút)

Bài hiện tại đã có SSH key và GitHub Secrets. Chỉ khi thiết lập máy/repo mới, tạo key chưa tồn tại bằng `ssh-keygen -t ed25519 -f deploy_key`, thêm **public key** vào `~/.ssh/authorized_keys` trên server. Thêm ba Secrets trong GitHub → Settings → Secrets and variables → Actions:

| Secret | Giá trị |
|---|---|
| SERVER_HOST | 221.121.5.186 |
| SERVER_USER | root |
| SERVER_SSH_KEY | Toàn bộ private key deploy_key |

Copy private key vào clipboard, không dán vào chat/README:

```powershell
Get-Content -Raw .\deploy_key | Set-Clipboard
```

Với bài đã cấu hình, chạy:

```powershell
git check-ignore deploy_key
git add Dockerfile docker-compose.yml build.gradle settings.gradle student.env nginx src .github README.md
git diff --cached --name-only
git commit -m "Complete application and deployment"
git push origin main
Start-Process "https://github.com/embedangyeunhathemattroi/devops/actions"
```

Nếu Git báo không có thay đổi thì bỏ qua commit, không coi đó là lỗi. `deploy_key` phải được ignore và không xuất hiện trong staged files. Không force push. Chờ **Build & Test** và **Deploy lên server** đều xanh. CI tự build, kiểm tra 8 lab/thông tin sinh viên/SHA, push image SHA + latest và deploy image; không build lại trên server. `settings.gradle` đã bổ sung Maven Central để tránh lỗi tải dependency qua Plugin Portal.

## 6. Kiểm tra server và đường truy cập ngoài (30–40 phút)

Trong PowerShell:

```powershell
ssh -i .\deploy_key root@221.121.5.186
```

Trong Linux server:

```bash
cd ~/devops
git rev-parse HEAD
docker compose --env-file student.env ps
curl -fsS --retry 12 --retry-delay 5 --retry-all-errors --max-time 10 http://127.0.0.1:8089/api/labs
curl -fsS --max-time 10 http://127.0.0.1:8089/api/info
exit
```

Trở về PowerShell:

```powershell
curl.exe -fsS --max-time 10 http://221.121.5.186:8089/api/labs
Start-Process "http://221.121.5.186:8089/student.html"
```

Phải thấy 8 phòng lab, đúng sinh viên và SHA trùng run Actions. API chạy trong server nhưng ngoài máy timeout là vấn đề cổng/firewall/security group; mở TCP **8089** trong bảng quản trị nhà cung cấp server hoặc nhờ quản trị mạng. Không reset MySQL để sửa lỗi mạng. Kiểm tra public là bước riêng, pipeline xanh không tự chứng minh cổng bên ngoài đã mở.

Nếu cần xem tạm bản server khi cổng public chưa mở, mở terminal thứ hai:

```powershell
ssh -N -L 18089:127.0.0.1:8089 -i .\deploy_key root@221.121.5.186
```

Giữ terminal đó chạy, mở `http://localhost:18089/student.html`. Đây là **server qua SSH tunnel**, không thay thế minh chứng truy cập trực tiếp IP public. Ctrl+C để đóng tunnel.

## 7. Xử lý lỗi trong vài phút

Đọc log đúng project trước, không thử xóa volume:

```powershell
docker compose --env-file student.env ps
docker compose --env-file student.env logs --tail 80 app mysql nginx
```

| Triệu chứng | Cách xử lý đúng |
|---|---|
| Docker engine chưa chạy | Bật Docker Desktop, chạy lại `docker version` |
| MySQL unhealthy / Access denied | Dừng sửa env tùy ý; tài khoản volume và env đang lệch. Nhờ quản trị dùng mật khẩu root hiện có, sao lưu rồi `ALTER USER` riêng app_user theo env; giữ dữ liệu |
| 502 ngay sau deploy | Chờ app khởi động, retry curl; nếu còn lỗi đọc log app |
| App đã chạy nhưng Nginx 502 | `docker compose --env-file student.env restart nginx` |
| Port already allocated | Kiểm tra chủ dịch vụ, không dừng container người khác; chọn cổng riêng và cập nhật env/URL/firewall đồng bộ |
| Build không tải dependency | Xem log đầu tiên, kiểm tra mạng; Maven Central đã cấu hình. Lỗi tải tạm thời thì Re-run failed jobs |
| SSH Permission denied | Kiểm tra đúng private key, public key trên server, SERVER_USER/HOST/SSH_KEY |
| Curl trong server được, ngoài máy timeout | Kiểm tra security group/firewall TCP 8089 tại nhà cung cấp |

Kiểm tra xác thực MySQL thực sự (chạy **trong Linux server**, không dán trực tiếp vào PowerShell):

```bash
cd ~/devops
docker compose --env-file student.env exec -T mysql sh -c 'MYSQL_PWD="$MYSQL_PASSWORD" mysql -h 127.0.0.1 -u"$MYSQL_USER" -D"$MYSQL_DATABASE" -Nse "SELECT COUNT(*) FROM labs"'
```

Kết quả phải là `8`. Sau khi mật khẩu đã được sửa trong MySQL, chỉ cần restart app/nginx; không xóa database. Nếu root trong env không đăng nhập được, cần dùng mật khẩu quản trị thật đã lưu từ lúc tạo volume; không suy ra rằng database hỏng.

## 8. Minh chứng và nộp bài (40–60 phút)

Bốn ảnh bạn cung cấp được giữ nguyên trong `screenshots/user-provided/`: ba ảnh thuộc repo cũ/SHA `0a2a190...`, ảnh website là localhost/SHA `3cfb43e...`. Chúng là tư liệu tham khảo, không chứng minh bản triển khai hiện tại. Xem `submission-content/verification.txt` cho kết quả kiểm tra và giới hạn thực tế.

Chụp bốn ảnh mới cùng một lần triển khai:

| File trong screenshots/ | Nội dung |
|---|---|
| 01-actions.png | Run repo hiện tại trên main, hai job xanh |
| 02-ghcr.png | Package repo hiện tại có latest và SHA của run đó |
| 03-compose-ps.png | Terminal SSH server, đúng project và image SHA |
| 04-website.png | Browser có IP server:8089, sinh viên/SHA/8 phòng lab |

Không sửa nội dung ảnh để giả thành IP/SHA khác. Ảnh qua tunnel phải ghi rõ tunnel; ảnh localhost không thay thế ảnh public.

Sau khi lưu ảnh đúng tên, chạy PowerShell tại thư mục dự án:

```powershell
New-Item -ItemType Directory -Force submission-content | Out-Null
Copy-Item screenshots\*.png submission-content\
git add README.md screenshots submission-content
git commit -m "Add submission evidence"
git push origin main
```

Chờ run cuối xanh. Ảnh là bằng chứng của SHA được chụp trước commit ảnh; giữ link run đó trong hồ sơ, không sửa SHA trên ảnh. Nộp link repo và thư mục minh chứng lên RAIA/LMS bằng tài khoản sinh viên; push GitHub không tự nộp LMS. Giữ server chạy đến khi chấm.

Nếu đề yêu cầu tải thư mục từ server bằng SFTP, dùng terminal IntelliJ sau khi server đã cập nhật:

```powershell
New-Item -ItemType Directory -Force ..\bai-nop | Out-Null
Set-Location ..\bai-nop
sftp -i "C:\Users\TRONG TIN\Downloads\devops-final-app-de001\devops-final-app-de001\deploy_key" root@221.121.5.186
```

Trong `sftp>`:

```text
get -r devops
bye
```

Không đưa private key hoặc bản sao lưu database vào gói nộp.
