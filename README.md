# Calling Messenger Hub cho ZimaOS / Docker

Bản Docker độc lập của **Calling Messenger Hub**, dùng được trên ZimaOS, NAS hoặc máy chủ Linux có Docker.

Repo này chỉ chứa tệp triển khai. Mã nguồn không được đưa vào repo; người dùng chạy image đã build sẵn từ GHCR.

- Image: `ghcr.io/trankhanhduy2929-beep/calling_messenger_hub`
- Kiến trúc: `amd64`, `arm64`
- Phiên bản trong compose: `0.22.1`
- Cổng mặc định: `3030`
- Dữ liệu lâu dài: thư mục `./data`

## Có gì mới trong 0.22.1

- Toàn bộ cấu hình cần thiết nằm trong **Cài đặt** của giao diện; không cần dùng tab **Configuration** của add-on.
- Thêm mục **Mặc định add-on** để chỉnh giọng/tốc độ TTS, thời gian chờ, số lần gọi và giới hạn cuộc gọi.
- License key nhập trong giao diện; license server/site URL tự dùng mặc định.
- Giữ nguyên dữ liệu MQTT và mặc định đang nhập khi trang tự làm mới.

## Yêu cầu

- ZimaOS hoặc máy Linux có Docker Engine và Docker Compose.
- Kết nối Internet cho Telegram, Zalo, TTS và license.
- Nếu dùng SIM 4G: modem USB xuất hiện trên host dưới dạng `/dev/ttyUSB*` hoặc `/dev/ttyACM*`.
- Không cần clone mã nguồn.

## Cài trên ZimaOS

1. Tạo thư mục ứng dụng, ví dụ:

   ```sh
   mkdir -p /DATA/AppData/calling-messenger-hub/data
   cd /DATA/AppData/calling-messenger-hub
   ```

2. Tải `docker-compose.yml` và `.env.example` vào thư mục này.

3. Tạo tệp cấu hình:

   ```sh
   cp .env.example .env
   chmod 600 .env
   ```

4. Mở `.env`, đặt `CMH_API_TOKEN` bằng một chuỗi ngẫu nhiên mạnh. Có thể tạo bằng:

   ```sh
   openssl rand -hex 32
   ```

5. Trong ZimaOS, mở phần **Docker / Compose**, chọn nhập hoặc tạo ứng dụng từ `docker-compose.yml`, rồi đặt thư mục dự án là `/DATA/AppData/calling-messenger-hub`.

6. Khởi động ứng dụng và mở:

   ```text
   http://IP-CUA-ZIMAOS:3030
   ```

7. Nhập API token vừa đặt trong `.env` để đăng nhập giao diện.

Nếu ZimaOS không tự đọc `.env`, nhập các biến môi trường tương ứng trong phần cấu hình ứng dụng.

Nếu ZimaOS không nhận đường dẫn tương đối `./data`, đổi volume trong compose thành đường dẫn tuyệt đối:

```yaml
volumes:
  - "/DATA/AppData/calling-messenger-hub/data:/data"
```

## Cài bằng Docker CLI

```sh
git clone https://github.com/trankhanhduy2929-beep/calling_messenger_zimaos.git
cd calling_messenger_zimaos
cp .env.example .env
chmod 600 .env
```

Sửa `.env`, sau đó chạy:

```sh
docker compose pull
docker compose up -d
docker compose logs -f
```

Kiểm tra dịch vụ:

```sh
curl http://127.0.0.1:3030/health
```

Kết quả hợp lệ có dạng:

```json
{"status":"ok","version":"0.22.1"}
```

## API token

Có hai cách lấy token:

- Đặt `CMH_API_TOKEN` trong `.env` trước khi khởi động.
- Để trống, ứng dụng sẽ tự tạo token và lưu tại `data/api_token`.

Đọc token tự sinh:

```sh
cat data/api_token
```

Token này dùng cho giao diện web, REST API và custom integration Home Assistant. Không đăng token lên mạng xã hội, ảnh chụp màn hình hoặc kho mã công khai.

## Kết nối Home Assistant

1. Cài custom integration `calling_messenger_hub` từ repo:

   ```text
   https://github.com/trankhanhduy2929-beep/calling_messenger_hub
   ```

2. Thêm integration **Calling Messenger Hub**.
3. Nhập URL của bản Docker, ví dụ:

   ```text
   http://192.168.1.50:3030
   ```

4. Nhập API token ở trên.

Bản Docker không dùng ingress của Home Assistant. Nếu Home Assistant và Docker chạy trên cùng máy, có thể dùng `http://127.0.0.1:3030` hoặc IP LAN của máy đó.

## Dùng SIM 4G qua USB

Chỉ cần bật thiết bị USB khi bạn muốn dùng modem SIM. Không dùng SIM thì bỏ qua phần này.

Kiểm tra thiết bị trên host:

```sh
ls -l /dev/ttyUSB* /dev/ttyACM*
```

Chạy kèm tệp cấu hình thiết bị:

```sh
docker compose -f docker-compose.yml -f docker-compose.sim.yml up -d
```

Mặc định ánh xạ `/dev/ttyUSB2`. Có thể đổi trong `.env`:

```dotenv
CMH_SIM_HOST_DEVICE=/dev/ttyUSB2
CMH_SIM_CONTAINER_DEVICE=/dev/ttyUSB2
```

Sau đó vào giao diện web, thêm tài khoản SIM:

- Kiểu kết nối: `Serial / USB (UART)`
- Cổng điều khiển: đúng đường dẫn trong container, thường là `/dev/ttyUSB2`
- Cổng sự kiện: để trống nếu chỉ có một cổng; với EC20/EC25 hai cổng AT, dùng cổng còn lại như `/dev/ttyUSB3`
- Baudrate: `115200`
- Driver: `Tự động`, `A7670` hoặc `EC20`

Nếu dùng cổng sự kiện riêng, ánh xạ thêm thiết bị trong `docker-compose.sim.yml`:

```yaml
services:
  calling-messenger-hub:
    devices:
      - "${CMH_SIM_HOST_DEVICE:-/dev/ttyUSB2}:${CMH_SIM_CONTAINER_DEVICE:-/dev/ttyUSB2}"
      - "/dev/ttyUSB3:/dev/ttyUSB3"
```

Không phải firmware nào cũng tách URC ra cổng thứ hai; kiểm tra bằng **Quét cổng USB** và để trống cổng sự kiện nếu không chắc.

### Nạp driver trên host

Bản Docker không tự nạp kernel module như add-on Home Assistant. Nếu host chưa có cổng tty cho modem:

```sh
lsmod | grep -E 'option|usbserial'
sudo modprobe option
sudo modprobe usbserial
```

Sau đó rút và cắm lại modem USB, rồi kiểm tra lại `/dev/ttyUSB*`.

Không nhập đường dẫn `/dev/bus/usb/...` trong bản Docker. Hãy dùng cổng `/dev/ttyUSB*` hoặc `/dev/ttyACM*` đã được ánh xạ vào container.

Nếu ZimaOS chạy trong máy ảo, cần passthrough modem USB vào máy ảo trước.

## Giám sát và MQTT

- Mở **Cài đặt → MQTT & Home Assistant** trong giao diện để bật MQTT Discovery. Add-on tự tạo cảm biến cho tài khoản online, cuộc gọi, cảnh báo chưa xác nhận, sức khỏe SIM, bản cập nhật và cường độ sóng từng SIM.
- Prometheus scrape `http://IP-CUA-ZIMAOS:3030/metrics`. Endpoint không yêu cầu token; chỉ mở cổng 3030 trong LAN tin cậy hoặc đặt sau reverse proxy có xác thực.
- Trang **SIM & số dư** có badge sức khỏe tổng hợp; nút **Kiểm tra cập nhật** trong **Cài đặt** chỉ báo phiên bản mới, không tự nâng cấp container.

## Biến môi trường

| Biến | Mặc định | Mô tả |
| --- | --- | --- |
| `CMH_VERSION` | `0.22.1` | Tag image cần chạy |
| `CMH_BIND_IP` | `0.0.0.0` | IP host dùng để publish cổng |
| `CMH_PORT` | `3030` | Cổng web/API trên host |
| `TZ` | `Asia/Ho_Chi_Minh` | Múi giờ |
| `CMH_API_TOKEN` | tự sinh | Token web/API |
| `TGVOIP_API_ID` | trống | Telegram API ID |
| `TGVOIP_API_HASH` | trống | Telegram API hash |
| `TGVOIP_LICENSE_KEY` | trống | License key |
| `TGVOIP_TTS_VOICE` | `vi-VN-HoaiMyNeural` | Giọng TTS mặc định |
| `TGVOIP_TTS_RATE` | `+0%` | Tốc độ TTS |
| `TGVOIP_TTS_CACHE` | `true` | Bật cache TTS |
| `CMH_SIM_HOST_DEVICE` | `/dev/ttyUSB2` | Thiết bị serial trên host |
| `CMH_SIM_CONTAINER_DEVICE` | `/dev/ttyUSB2` | Thiết bị serial trong container |

Các biến nâng cao có thể đặt thêm nếu cần:

| Biến | Mặc định |
| --- | --- |
| `TGVOIP_RING_TIMEOUT` | `45` |
| `TGVOIP_RING_ATTEMPTS` | `2` |
| `TGVOIP_RING_INTERVAL` | `15` |
| `TGVOIP_MAX_CALL_SECONDS` | `180` |
| `TGVOIP_LOG_LIMIT` | `300` |
| `TGVOIP_LICENSE_SERVER_URL` | license server mặc định |
| `TGVOIP_LICENSE_SITE_URL` | license site mặc định |

## Dữ liệu và sao lưu

Thư mục `data/` chứa:

- tài khoản Telegram, Discord, Signal, LINE, Zalo và SIM;
- phiên đăng nhập;
- cache TTS;
- cấu hình automation;
- API token tự sinh.

Sao lưu toàn bộ thư mục `data/` trước khi nâng cấp hoặc chuyển máy. Không chia sẻ thư mục này vì có thể chứa phiên đăng nhập và token.

Trong giao diện, mở **Cài đặt → Sao lưu & khôi phục** để tạo tệp `.cmhb` được mã hóa AES-256-GCM bằng mật khẩu riêng. Tệp này chỉ chứa cấu hình, tài khoản, phiên đăng nhập và token cần thiết; không chứa cache TTS. Khôi phục sẽ ghi đè tệp trùng tên và yêu cầu khởi động lại container. Nếu quên mật khẩu, không thể giải mã bản sao lưu.

## Cập nhật

Đổi `CMH_VERSION` trong `.env` nếu cần, sau đó:

```sh
docker compose pull
docker compose up -d --remove-orphans
```

Với cấu hình có SIM:

```sh
docker compose -f docker-compose.yml -f docker-compose.sim.yml pull
docker compose -f docker-compose.yml -f docker-compose.sim.yml up -d --remove-orphans
```

## Bảo mật

- Đặt API token dài và ngẫu nhiên.
- Không publish cổng ra Internet trực tiếp.
- Nếu dùng reverse proxy, đặt `CMH_BIND_IP=127.0.0.1`, bật HTTPS và xác thực bổ sung.
- Chỉ cấp quyền truy cập cho người dùng tin cậy.
- Dùng tài khoản phụ cho Telegram/Zalo và tuân thủ điều khoản nền tảng.
- Không commit tệp `.env` hoặc thư mục `data/`.

## Xử lý sự cố

### 401 invalid API token

Token nhập vào giao diện không khớp. Kiểm tra `CMH_API_TOKEN`, hoặc đọc `data/api_token` nếu không đặt biến môi trường. Sau khi đổi token, khởi động lại container.

### Không mở được cổng 3030

Kiểm tra cổng đã bị chiếm:

```sh
ss -ltnp | grep 3030
```

Đổi `CMH_PORT` trong `.env` rồi chạy lại compose.

### Không thấy modem SIM

- Kiểm tra `/dev/ttyUSB*` trên host.
- Kiểm tra modem đã được passthrough vào ZimaOS/máy ảo.
- Kiểm tra driver `option`/`usbserial`.
- Kiểm tra tệp `docker-compose.sim.yml` đã được dùng khi khởi động.
- Xem log: `docker compose logs -f`.

### License không hợp lệ

Nhập key trong giao diện **Cài đặt > License** hoặc đặt `TGVOIP_LICENSE_KEY` trong `.env` rồi khởi động lại.

### Xem log

```sh
docker compose logs -f --tail=200
```

## Gỡ cài đặt

```sh
docker compose down
```

Lệnh trên không xóa `data/`. Chỉ xóa thư mục `data/` khi chắc chắn không cần dữ liệu nữa.
