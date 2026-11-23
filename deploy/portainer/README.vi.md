# Triển khai Stirling-PDF trên Portainer (Proxmox VM/LXC, chỉ mạng LAN)

**Nguồn:** [fork hoangthang0106/Stirling-PDF](https://github.com/hoangthang0106/Stirling-PDF). **File Stack:** [stirling-pdf.yml](./stirling-pdf.yml).

## Các quyết định cấu hình

| Mục | Giá trị |
|---|---|
| Host port mặc định | **18993/TCP** (CHƯA kiểm tra trực tiếp trên máy) |
| Port bên trong container | **8080/TCP** |
| IP bind | `LAN_IP` — **nhập tay**, IP thật của VM/LXC chạy Docker |
| Tên container | `stirling-pdf` |
| Image | `stirlingtools/stirling-pdf:3.5.0` (image chính thức từ Docker Hub, không build từ fork) |
| Đăng nhập | **Bật** |
| Tên đăng nhập khởi tạo | `admin` mặc định, đổi bằng `STIRLING_ADMIN_USER` |
| Mật khẩu | **Nhập tay** qua `STIRLING_ADMIN_PASSWORD`, KHÔNG commit GitHub |
| Ngôn ngữ | Tiếng Việt `vi-VN` |
| Volume | Sáu named volumes để giữ cài đặt, tài khoản, log và dữ liệu |

**Phân biệt:** File Compose đặt trong fork của bạn. Image chạy là image **chính thức** (không phải image tự build từ mã nguồn fork). Cách này tránh build repo lớn trên VM/LXC và tránh lỗi kéo image riêng GHCR đã từng gặp.

## Bước 1 — Kiểm kê chính xác, chỉ đọc

Chạy **trong VM/LXC đang chạy Portainer/Docker** (không chạy trên Proxmox host, trừ khi Docker thực sự nằm trên host):

```bash
hostname
hostname -I
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker compose ls -a
sudo ss -lntup
sudo ss -ltnp '( sport = :18993 )'
docker ps -aq | xargs -r docker inspect --format '{{.Name}} {{json .HostConfig.PortBindings}}'
```

**Chưa được kết luận 18993 trống** chỉ vì hồ sơ cũ không có port này. Xem kết quả của `ss` cùng các host-port trong `docker inspect`, kể cả container đang dừng. Kiểm tra xung đột theo *cùng Docker host + IP bind + TCP/UDP + host port*. IP của Proxmox host và VM/LXC có thể khác nhau.

Nếu `18993` trùng, tự chọn một port thật sự còn trống và nhập biến `STIRLING_HOST_PORT` khi triển khai. Không dừng hay xóa ứng dụng cũ để lấy port.

## Bước 2 — Triển khai trong Portainer từ GitHub fork

1. Mở **Portainer → Environments → đúng Docker endpoint → Stacks → Add stack**.
2. Đặt tên Stack: `stirling-pdf`.
3. Chọn **Repository** / **Git repository** (không dùng build repo của Stirling).
4. Repository URL: `https://github.com/hoangthang0106/Stirling-PDF`
5. Reference: `refs/heads/main` (hoặc `main` theo giao diện Portainer).
6. Compose path: `deploy/portainer/stirling-pdf.yml` (đường dẫn tương đối từ gốc repo).
7. Tại phần **Environment variables**, nhập **thủ công**:
   - `LAN_IP`: IP LAN thật của VM/LXC chạy Docker, ví dụ từ `hostname -I` sau khi xác nhận.
   - `STIRLING_ADMIN_PASSWORD`: mật khẩu mạnh, duy nhất; không gửi qua chat, không commit Git.
   - **Tùy chọn** `STIRLING_HOST_PORT`: chỉ nhập nếu 18993 trùng, giá trị phải là port TCP trống đã kiểm tra.
   - **Tùy chọn** `STIRLING_ADMIN_USER`: username riêng thay `admin`.
8. Kiểm tra tất cả giá trị và **chỉ khi chủ động xác nhận**, nhấn **Deploy the stack**.

> Chú ý: các biến bí mật nhập vào Portainer sẽ nằm trong môi trường của container/Stack và có thể được người quản trị Docker/Portainer đọc. Không coi đây là vault bí mật; hạn chế quyền quản trị và xem xét secret manager khi cần.

## Bước 3 — Kiểm tra sau khi triển khai

- **Portainer → Containers → stirling-pdf → Logs**: bảo đảm không lỗi port, image, mount và không bị restart liên tục.
- Trên máy cùng LAN mở `http://LAN_IP:18993`, hoặc port bạn đã nhập trong `STIRLING_HOST_PORT`.
- Đăng nhập bằng `STIRLING_ADMIN_USER` (mặc định `admin`) và mật khẩu đã nhập **lần đầu**.
- Thông tin đăng nhập khởi tạo chỉ áp dụng khi database chưa có người dùng. Nếu đã có tài khoản, đổi biến môi trường không reset được tài khoản cũ.

## Bảo mật và sao lưu

- Không mở NAT/port-forward 18993 ra Internet; giới hạn bằng firewall LAN, VLAN/ACL nếu cần.
- Giao thức hiện tại là **HTTP**; không gửi tài liệu nhạy cảm hay thông tin đăng nhập qua mạng không tin cậy. Khi dùng dữ liệu thật, cấu hình HTTPS nội bộ (ví dụ reverse proxy Caddy với TLS phù hợp) trước khi mở rộng truy cập.
- Các Docker named volumes **không phải backup**. Sao lưu riêng, đặc biệt volume `/configs` chứa tài khoản và cấu hình, trước khi cập nhật image.
- Không chọn **Remove volumes** nếu cần giữ dữ liệu.
- Không chạy `docker compose down -v`, reset/xóa dữ liệu, hoặc thay đổi firewall mà chưa có xác nhận rõ ràng.

## Lưu ý vận hành

- Port `8080` bên trong Stirling-PDF không xung đột với các container khác cùng dùng port nội bộ 8080. Xung đột chỉ có thể xảy ra ở **host port** trên đúng IP và giao thức.
- Mã nguồn GitHub fork **không chứng minh ứng dụng đã được cài**; chỉ sau khi Portainer chạy thành công mới ghi nhận đã triển khai.
- Cấu hình này **không tự triển khai** và không tạo thay đổi trên VM/LXC hoặc NAS. GitHub chỉ giữ hướng dẫn và Compose.
