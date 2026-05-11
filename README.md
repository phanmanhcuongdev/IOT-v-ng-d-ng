# RoadmapForMe

RoadmapForMe là tài liệu roadmap học và triển khai dự án IoT/ESP32 theo hướng thực dụng cho người đã quen backend, homelab, Docker, OpenWrt, MQTT hoặc API nhưng còn mới với phần cứng nhúng.

Tài liệu chính hiện nằm tại [roadmap.md](roadmap.md).

## Nội dung chính

- Chiến lược học ESP32 theo từng bước nhỏ.
- Cách kiểm thử phần cứng theo nguyên tắc atomic testing.
- Lộ trình làm quen Embedded C++, PlatformIO và Arduino Framework.
- Các mốc thực hành với GPIO, PIR, OLED, WiFi, MQTT và ESP32-CAM.
- Hướng tích hợp server-side AI bằng Python, FastAPI và OpenCV trên Proxmox.
- Các lỗi thường gặp khi làm phần cứng như brownout, sai GPIO, nguồn yếu hoặc nối dây không ổn định.

## Đối tượng phù hợp

Roadmap này phù hợp nếu bạn đã có nền tảng về:

- Backend development
- Proxmox homelab
- Docker hoặc service deployment
- OpenWrt, VPN hoặc network routing
- MQTT, API hoặc backend integration

Và muốn bắt đầu với:

- Embedded C++
- ESP32
- Wiring phần cứng
- Sensor, camera, OLED
- Debug lỗi nguồn, GPIO và firmware

## Công cụ khuyến nghị

- IDE: VS Code
- Build tool: PlatformIO
- Framework: Arduino Framework
- Ngôn ngữ firmware: C++
- Server AI: Python, FastAPI, OpenCV
- Kiến trúc: Dumb Edge, Smart Cloud

## Cách sử dụng

1. Đọc [roadmap.md](roadmap.md) từ đầu đến cuối để nắm chiến lược tổng thể.
2. Thực hiện từng mốc nhỏ theo đúng thứ tự.
3. Khi gặp lỗi, chỉ thay đổi một biến tại một thời điểm: code, dây nối, GPIO, nguồn hoặc library.
4. Luôn kiểm tra bằng Serial Monitor trước khi ghép thêm OLED, MQTT, HTTP hoặc AI.

## Trạng thái dự án

Repository hiện là tài liệu roadmap. Nếu sau này bổ sung code PlatformIO, firmware ESP32 hoặc server AI, nên đặt chúng trong các thư mục riêng như:

- `firmware/`
- `server/`
- `docs/`

## License

Chưa khai báo license.
