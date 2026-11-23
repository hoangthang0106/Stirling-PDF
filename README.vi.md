# Stirling-PDF — triển khai từ fork của bạn bằng Portainer (LAN)

Nhánh **`portainer-clean`** chỉ chứa 2 tệp thường, **không chứa symlink**.
Portainer chặn symbolic links có trong mã nguồn `main`. Dùng nhánh này để tránh lỗi khi clone.
**Không cần xóa symlink hay sửa mã nguồn trên nhánh main.**

## Tại Portainer → Stacks → Add stack → Repository

| Trường | Nhập |
|---|---|
| Name | `stirling-pdf` |
| Repository URL | `https://github.com/hoangthang0106/Stirling-PDF` |
| Repository reference | `refs/heads/portainer-clean` |
| Compose path | `docker-compose.yml` |

Ở **Environment variables** nhập thủ công:
- `LAN_IP`: IP LAN **thực tế** của VM/LXC chạy Docker/Portainer (không mặc định là IP Proxmox host).
- `STIRLING_ADMIN_PASSWORD`: mật khẩu mạnh, riêng; **không nhập vào GitHub hoặc chat**.
- `STIRLING_HOST_PORT` *(tùy chọn)*: mặc định `18993`; chỉ đổi khi đã kiểm tra 18993 trùng.
- `STIRLING_ADMIN_USER` *(tùy chọn)*: mặc định `admin`.

## Kiểm tra trước khi nhấn Deploy

Chạy trên **chính VM/LXC chạy Docker**. Các lệnh dưới đây chỉ đọc:

```bash
hostname -I
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
sudo ss -ltnp '( sport = :18993 )'
docker ps -aq | xargs -r docker inspect --format '{{.Name}} {{json .HostConfig.PortBindings}}'
```

Không được suy luận cổng 18993 trống từ danh sách port cũ. Phải kiểm tra port host TCP theo IP bind,
kể cả Docker container đang dừng. Nếu port có người dùng, đặt biến `STIRLING_HOST_PORT` bằng cổng khác được xác minh trống.
Không dừng/xóa dịch vụ có sẵn để giải phóng port.

## Triển khai và kiểm tra

1. Chỉ **Deploy the stack** sau khi kiểm tra IP, port và biến.
2. Vào Portainer → **Containers → stirling-pdf → Logs**, kiểm tra container `Running`.
3. Mở `http://LAN_IP:18993` (hoặc thay 18993 bằng port bạn tự đặt).
4. Đăng nhập bằng username/mật khẩu khởi tạo đã cấu hình. Nếu dữ liệu người dùng đã tồn tại, đặt lại biến không reset được mật khẩu cũ.

## Bảo mật & dữ liệu

- Image: `stirlingtools/stirling-pdf:3.5.0` (official Docker Hub prebuilt), không dùng Docker build.
- `8080` **bên trong container**; `18993` **host port** chỉ là mặc định chưa xác minh.
- `LAN_IP` bind theo địa chỉ LAN được nhập; không cấu hình port-forward/NAT ra Internet.
- Cấu hình bật đăng nhập và ngôn ngữ tiếng Việt `vi-VN`.
- Mật khẩu trong biến môi trường vẫn có thể đọc bởi quản trị viên Docker/Portainer.
- HTTP không mã hóa; dùng HTTPS nội bộ khi cần xử lý tài liệu hoặc tài khoản nhạy cảm.
- Named volumes lưu dữ liệu qua quá trình tạo lại container nhưng không thay thế sao lưu.
- Không xóa volume hoặc tự chạy lệnh trên hệ thống khi chưa chủ động xác nhận.
