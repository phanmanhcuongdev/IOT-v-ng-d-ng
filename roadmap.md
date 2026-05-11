# Roadmap v1.1 — Hardware Survival Edition

## Mục tiêu

Bản này là phiên bản đã chỉnh lại để tránh lỗi parser: toàn bộ nội dung nằm trong **một code block duy nhất** và không dùng nested triple-backticks bên trong.

Roadmap này dành cho người đã mạnh về:

- Backend development
- Proxmox homelab
- Docker / service deployment
- OpenWrt / VPN / network routing
- MQTT / API / backend integration

Nhưng còn mới với:

- Embedded C++
- ESP32
- Wiring phần cứng
- Sensor / camera / OLED
- Lỗi nguồn, lỗi GPIO, lỗi brownout

---

# 1. Chiến lược tổng thể

## Quyết định kỹ thuật chính

| Hạng mục | Quyết định |
|---|---|
| IDE | VS Code |
| Build tool | PlatformIO |
| Framework | Arduino Framework |
| Ngôn ngữ | C++ |
| ESP-IDF | Không dùng ở giai đoạn đầu |
| Rust | Không dùng trong roadmap này |
| TinyML trên ESP32 | Không dùng |
| AI strategy | Dumb Edge, Smart Cloud |
| Server AI | Chạy trên Proxmox bằng Python / FastAPI / OpenCV |

---

# 2. Nguyên tắc sinh tồn cho hardware newbie

## 2.1. Atomic Testing

Không ghép nhiều linh kiện ngay từ đầu.

Thứ tự bắt buộc:

1. Blink LED
2. PIR to Serial
3. OLED Hello World
4. PIR + OLED
5. WiFi healthcheck
6. MQTT heartbeat
7. PIR event to MQTT
8. ESP32-CAM local test
9. ESP32-CAM HTTP upload
10. Server-side AI processing

---

## 2.2. One variable at a time

Khi lỗi, chỉ đổi một thứ tại một thời điểm:

| Có thể đổi | Không nên đổi cùng lúc |
|---|---|
| GPIO pin | Vừa đổi GPIO vừa đổi code vừa đổi nguồn |
| Nguồn cấp | Vừa đổi nguồn vừa đổi thư viện |
| Dây nối | Vừa đổi dây vừa đổi board |
| Library version | Vừa đổi library vừa đổi framework |

---

## 2.3. Serial first

Trước khi dùng OLED, MQTT, HTTP hoặc AI, luôn kiểm tra bằng Serial Monitor.

Ví dụ log nên có:

    [BOOT] Firmware started
    [PIR] MOTION
    [OLED] Init OK
    [WIFI] Connected
    [MQTT] Published
    [HTTP] Upload success

---

# 3. Ba bẫy phần cứng cần ghim ngay

## Trap 1: ESP32-CAM Brownout vì nguồn yếu

ESP32-CAM rất dễ reset nếu nguồn yếu. Khi WiFi bật, camera khởi động hoặc flash LED sáng, board có thể hút dòng tăng vọt trong thời gian rất ngắn.

### Dấu hiệu thường gặp

    Brownout detector was triggered

Hoặc:

    Camera init failed
    Guru Meditation Error
    Reset loop

### Cách cấp nguồn khuyến nghị

| Nguồn cấp | Đánh giá |
|---|---|
| USB hub rẻ tiền | Không nên dùng |
| Cổng USB màn hình | Không nên dùng |
| Cổng USB trước case PC | Có thể yếu |
| Cổng USB sau mainboard PC | Nên dùng khi test |
| Củ sạc điện thoại 5V-2A tốt | Rất nên dùng |
| Breadboard MB-102 power module | Cẩn thận, có thể không ổn định cho ESP32-CAM |

### Quy tắc thực chiến

- Khi test ESP32-CAM, ưu tiên cắm trực tiếp vào cổng USB sau mainboard.
- Nếu dùng nguồn ngoài, dùng nguồn 5V-2A tử tế.
- Không bật flash LED trong bài test đầu tiên.
- Nếu reset ngẫu nhiên, nghi nguồn trước khi nghi code.
- Nếu dùng nguồn ngoài, luôn nối chung GND.

---

## Trap 2: PIR HC-SR501 nên cấp VCC 5V

HC-SR501 thường chạy ổn nhất khi cấp VCC 5V. Module có mạch ổn áp nội bộ. Nếu cấp 3.3V, nhiều module sẽ bị ghost trigger, lúc nhận lúc không hoặc hoạt động thiếu ổn định.

### Pinout khuyến nghị

| PIR HC-SR501 | ESP32-S3 |
|---|---|
| VCC | 5V / VIN |
| GND | GND |
| OUT | GPIO 4 |

### Ghi chú an toàn

- OUT của HC-SR501 thường là logic 3.3V, an toàn cho GPIO ESP32.
- Không đưa tín hiệu 5V trực tiếp vào GPIO ESP32.
- Nếu module lạ, dùng đồng hồ đo điện áp OUT trước khi cắm vào ESP32.
- PIR cần warm-up vài giây đến vài chục giây sau khi cấp nguồn.

### Triệu chứng PIR không ổn định

| Hiện tượng | Nguyên nhân có thể |
|---|---|
| Tự báo motion dù không có ai | Nguồn không ổn định, sensitivity cao |
| Không báo motion | Wiring sai, cấp 3.3V yếu |
| Báo lúc được lúc không | Dây lỏng, nguồn yếu, chưa warm-up |
| Giữ HIGH quá lâu | Biến trở delay chỉnh quá cao |

---

## Trap 3: `delay()` là kẻ thù từ Phase 2 trở đi

Ở Phase 1, dùng `delay()` để blink LED hoặc test PIR là chấp nhận được.

Nhưng từ Phase 2, khi có WiFi / MQTT, không nên dùng `delay()` dài trong `loop()`.

### Vì sao?

Trong Arduino, dòng sau sẽ làm firmware gần như đứng chờ 5 giây:

    delay(5000);

Trong lúc đó:

- MQTT client không được gọi `mqttClient.loop()`.
- Reconnect logic không chạy.
- Sensor polling bị trễ.
- OLED không update kịp.
- WiFi/MQTT dễ timeout.
- Cảm giác như mạng chập chờn nhưng thật ra firmware đang bị block.

Nó không giống `await Task.Delay()` trong .NET.

### Không nên viết ở Phase 2

    void loop() {
      readPir();
      publishMqtt();
      delay(5000);
    }

### Nên chuyển sang `millis()`

    unsigned long lastHeartbeatMs = 0;
    unsigned long lastPirReadMs = 0;

    const unsigned long HEARTBEAT_INTERVAL_MS = 5000;
    const unsigned long PIR_READ_INTERVAL_MS = 200;

    void loop() {
      if (!mqttClient.connected()) {
        connectMQTT();
      }

      mqttClient.loop();

      unsigned long now = millis();

      if (now - lastHeartbeatMs >= HEARTBEAT_INTERVAL_MS) {
        lastHeartbeatMs = now;
        publishHeartbeat();
      }

      if (now - lastPirReadMs >= PIR_READ_INTERVAL_MS) {
        lastPirReadMs = now;
        readPirAndMaybePublish();
      }
    }

---

# 4. Phase 1 — The First Light

## Mục tiêu

Làm chủ phần cứng cơ bản bằng PlatformIO + Arduino Framework.

Không làm:

- WiFi
- MQTT
- HTTP
- AI
- ESP-IDF
- Rust
- TinyML

---

## 4.1. Toolchain Phase 1

| Thành phần | Dùng |
|---|---|
| IDE | VS Code |
| Build tool | PlatformIO |
| Framework | Arduino Framework |
| Language | C++ |
| Debug | Serial Monitor |
| OLED library | Adafruit SSD1306 + Adafruit GFX |

---

## 4.2. Step 1 — Blink LED

### Hardware

| Hardware | Dùng |
|---|---|
| ESP32-S3 DevKitC N16R8 | Có |
| PIR | Không |
| OLED | Không |
| ESP32-CAM | Không |

### Mục tiêu

- Board nhận COM port.
- PlatformIO upload được firmware.
- Serial Monitor chạy được.
- LED hoặc log blink đều.

### Code mẫu

    #include <Arduino.h>

    #ifndef LED_BUILTIN
    #define LED_BUILTIN 2
    #endif

    void setup() {
      Serial.begin(115200);
      delay(1000);

      pinMode(LED_BUILTIN, OUTPUT);
      Serial.println("ESP32-S3 Blink test started");
    }

    void loop() {
      digitalWrite(LED_BUILTIN, HIGH);
      Serial.println("LED ON");
      delay(500);

      digitalWrite(LED_BUILTIN, LOW);
      Serial.println("LED OFF");
      delay(500);
    }

### Checklist pass

| Điều kiện | Trạng thái |
|---|---|
| Upload firmware thành công | `[ ]` |
| Serial Monitor có log | `[ ]` |
| Board không reset ngẫu nhiên | `[ ]` |
| Biết giữ BOOT khi upload fail | `[ ]` |

---

## 4.3. Step 2 — PIR to Serial

### Hardware

| Hardware | Dùng |
|---|---|
| ESP32-S3 | Có |
| PIR HC-SR501 | Có |
| OLED | Không |
| ESP32-CAM | Không |

### Pinout

| PIR HC-SR501 | ESP32-S3 |
|---|---|
| VCC | 5V / VIN |
| GND | GND |
| OUT | GPIO 4 |

### Code mẫu

    #include <Arduino.h>

    constexpr int PIR_PIN = 4;

    void setup() {
      Serial.begin(115200);
      delay(1000);

      pinMode(PIR_PIN, INPUT);
      Serial.println("PIR serial test started");
      Serial.println("Waiting for PIR warm-up...");
      delay(3000);
    }

    void loop() {
      int motion = digitalRead(PIR_PIN);

      if (motion == HIGH) {
        Serial.println("MOTION DETECTED");
      } else {
        Serial.println("IDLE");
      }

      delay(500);
    }

### Checklist pass

| Điều kiện | Trạng thái |
|---|---|
| Serial in ra MOTION / IDLE | `[ ]` |
| Đưa tay qua PIR thì đổi trạng thái | `[ ]` |
| PIR không ghost trigger quá nhiều | `[ ]` |
| VCC PIR đang cắm 5V / VIN | `[ ]` |
| GND chung với ESP32 | `[ ]` |

---

## 4.4. Step 3 — OLED Hello World

### Hardware

| Hardware | Dùng |
|---|---|
| ESP32-S3 | Có |
| OLED I2C 0.96 inch | Có |
| PIR | Không |
| ESP32-CAM | Không |

### Pinout

| OLED I2C | ESP32-S3 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| SDA | GPIO 8 |
| SCL | GPIO 9 |

### PlatformIO libraries

    lib_deps =
      adafruit/Adafruit SSD1306
      adafruit/Adafruit GFX Library

### Code mẫu

    #include <Arduino.h>
    #include <Wire.h>
    #include <Adafruit_GFX.h>
    #include <Adafruit_SSD1306.h>

    constexpr int I2C_SDA = 8;
    constexpr int I2C_SCL = 9;

    constexpr int SCREEN_WIDTH = 128;
    constexpr int SCREEN_HEIGHT = 64;

    Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

    void setup() {
      Serial.begin(115200);
      delay(1000);

      Wire.begin(I2C_SDA, I2C_SCL);

      if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
        Serial.println("OLED init failed");
        while (true) {
          delay(1000);
        }
      }

      display.clearDisplay();
      display.setTextSize(1);
      display.setTextColor(SSD1306_WHITE);
      display.setCursor(0, 0);
      display.println("Hello ESP32-S3");
      display.println("OLED OK");
      display.display();

      Serial.println("OLED hello world displayed");
    }

    void loop() {
    }

### Checklist pass

| Điều kiện | Trạng thái |
|---|---|
| OLED sáng | `[ ]` |
| Hiện chữ Hello ESP32-S3 | `[ ]` |
| Serial báo OLED OK | `[ ]` |
| Nếu lỗi, đã thử địa chỉ 0x3C / 0x3D | `[ ]` |

---

## 4.5. Step 4 — Combine PIR + OLED

### Hardware

| Hardware | Dùng |
|---|---|
| ESP32-S3 | Có |
| PIR HC-SR501 | Có |
| OLED I2C | Có |
| ESP32-CAM | Không |

### Pinout tổng hợp

| Module | Pin | ESP32-S3 |
|---|---|---|
| PIR | VCC | 5V / VIN |
| PIR | GND | GND |
| PIR | OUT | GPIO 4 |
| OLED | VCC | 3V3 |
| OLED | GND | GND |
| OLED | SDA | GPIO 8 |
| OLED | SCL | GPIO 9 |

### Kết quả mong muốn

| PIR state | OLED display |
|---|---|
| LOW | Status: IDLE |
| HIGH | Status: MOTION DETECTED |

---

## 4.6. Mini-project Phase 1 — Motion Status Panel

| Mục | Nội dung |
|---|---|
| Hardware | ESP32-S3 + PIR + OLED |
| Input | PIR motion signal |
| Output | OLED hiển thị trạng thái |
| Debug | Serial Monitor |
| Không làm | WiFi, MQTT, camera, AI |
| Kỹ năng đạt được | GPIO, I2C, wiring, Serial debug |

---

# 5. Phase 2 — The Nervous System

## Mục tiêu

Đưa dữ liệu từ phần cứng về Proxmox backend qua OpenWrt VPN.

Không làm:

- Rust
- ESP-IDF
- TinyML
- AI trên ESP32

Tập trung:

- WiFi
- MQTT
- JSON
- HTTP cơ bản
- Reconnect logic
- Non-blocking loop bằng `millis()`

---

## 5.1. Kiến trúc Phase 2

    [ESP32-S3 + PIR]
          |
          | WiFi
          v
    [OpenWrt Router / VPN Gateway]
          |
          | VPN tunnel
          v
    [Proxmox Homelab]
          |
          +--> [MQTT Broker: Mosquitto / RabbitMQ MQTT plugin]
          |
          +--> [Backend API: FastAPI / Spring Boot]
          |
          +--> [Logs / Database / Dashboard]

---

## 5.2. Toolchain Phase 2

| Thành phần | Dùng |
|---|---|
| PlatformIO | Có |
| Arduino Framework | Có |
| C++ | Có |
| ArduinoJson | Có |
| PubSubClient | Có |
| WiFi.h | Có |
| HTTPClient.h | Có |
| Rust | Không |
| ESP-IDF | Không |

---

## 5.3. Step 1 — WiFi Healthcheck

### Mục tiêu

ESP32-S3 kết nối được WiFi và in IP/RSSI ra Serial.

### Log mong muốn

    WiFi connected
    IP: 192.168.x.x
    RSSI: -50

### Checklist pass

| Điều kiện | Trạng thái |
|---|---|
| ESP32 lấy được IP | `[ ]` |
| RSSI ổn định | `[ ]` |
| Không reset ngẫu nhiên | `[ ]` |
| OpenWrt route tới Proxmox đúng | `[ ]` |

---

## 5.4. Step 2 — MQTT Heartbeat JSON

### Topic đề xuất

    aiot/esp32s3-node-01/heartbeat

### Payload mẫu

    {
      "device_id": "esp32s3-node-01",
      "event_type": "heartbeat",
      "uptime_ms": 123456,
      "rssi": -55
    }

### Checklist pass

| Điều kiện | Trạng thái |
|---|---|
| MQTT broker nhận message | `[ ]` |
| Payload là JSON hợp lệ | `[ ]` |
| Có reconnect khi MQTT rớt | `[ ]` |
| Không dùng delay dài | `[ ]` |
| Có gọi mqttClient.loop() đều | `[ ]` |

---

## 5.5. Step 3 — PIR Event to MQTT

### Topic đề xuất

    aiot/esp32s3-node-01/events/motion

### Payload mẫu

    {
      "device_id": "esp32s3-node-01",
      "event_type": "motion_detected",
      "motion": true,
      "uptime_ms": 89123,
      "rssi": -58
    }

### Key concepts

| Concept | Lý do |
|---|---|
| Cooldown | Tránh spam event do PIR giữ HIGH |
| `millis()` | Không block MQTT loop |
| Reconnect | WiFi/MQTT có thể rớt |
| Small payload | ESP32 không nên gửi JSON quá to |
| Serial log | Debug nhanh khi broker không nhận |

---

## 5.6. Step 4 — OLED Network Status

### Mục tiêu

OLED hiển thị trạng thái mạng để không phải lúc nào cũng cắm Serial.

| Network state | OLED text |
|---|---|
| WiFi disconnected | WIFI LOST |
| WiFi connected | WIFI OK |
| MQTT disconnected | MQTT LOST |
| MQTT connected | MQTT OK |
| Publish success | EVENT SENT |
| Publish fail | SEND FAILED |

---

## 5.7. Mini-project Phase 2.1 — PIR MQTT Security Sensor

| Mục | Nội dung |
|---|---|
| Hardware | ESP32-S3 + PIR |
| Transport | MQTT |
| Payload | JSON |
| Server | MQTT broker trên Proxmox |
| Gateway | OpenWrt VPN |
| Success | Motion event xuất hiện trong MQTT subscriber log |

---

## 5.8. Mini-project Phase 2.2 — OLED Network Status Display

| Mục | Nội dung |
|---|---|
| Hardware | ESP32-S3 + OLED |
| Input | WiFi/MQTT status |
| Output | OLED hiển thị trạng thái |
| Success | Rút broker/server thì OLED báo lỗi |
| Kỹ năng đạt được | Firmware state management |

---

# 6. Phase 3 — The Brain Expansion

## Mục tiêu

Xây dựng AIoT theo mô hình:

    Dumb Edge, Smart Cloud

ESP32 chỉ làm:

- Đọc PIR
- Chụp ảnh
- Gửi ảnh
- Nhận kết quả
- Hiển thị OLED

Proxmox làm:

- FastAPI nhận ảnh
- Lưu ảnh
- OpenCV preprocessing
- AI inference
- Trả kết quả

---

## 6.1. Kiến trúc Phase 3

    [PIR HC-SR501]
          |
          v
    [ESP32-S3] ---- MQTT motion event ----+
          |                               |
          |                               v
          |                         [FastAPI / MQTT Worker]
          |                               |
          v                               |
    [OLED Display] <---- result ----------+
          
    [ESP32-CAM]
          |
          | HTTP POST image
          v
    [FastAPI on Proxmox]
          |
          +--> Save image
          +--> Run OpenCV / AI model
          +--> Return result

---

## 6.2. Step 1 — ESP32-CAM Local Bring-up

### Không upload server vội

Chỉ test camera trước:

| Checklist | Trạng thái |
|---|---|
| Flash được ESP32-CAM | `[ ]` |
| Camera init thành công | `[ ]` |
| Chụp được ảnh JPEG | `[ ]` |
| Không brownout reset | `[ ]` |
| Không bật flash LED lúc đầu | `[ ]` |

### Lỗi phổ biến

| Lỗi | Nguyên nhân |
|---|---|
| Brownout detector was triggered | Nguồn yếu |
| Camera init failed | Sai board/pin config |
| Upload fail | Chưa vào boot mode |
| Ảnh đen | Ánh sáng yếu hoặc camera lỗi |
| Reset loop | Nguồn không đủ dòng |

---

## 6.3. Step 2 — ESP32-CAM HTTP Image Upload

### Endpoint đề xuất

    POST /api/v1/iot/images

### Data format

| Kiểu | Gợi ý |
|---|---|
| Image | JPEG binary |
| Upload | HTTP multipart |
| Metadata | device_id, timestamp |
| Không nên | Base64 ảnh trong JSON ở giai đoạn đầu |

### Checklist pass

| Điều kiện | Trạng thái |
|---|---|
| FastAPI nhận được ảnh | `[ ]` |
| Ảnh lưu được trên server | `[ ]` |
| File ảnh mở được | `[ ]` |
| Upload chạy qua OpenWrt VPN | `[ ]` |

---

## 6.4. Step 3 — Server-side Image Processing

### Pipeline xử lý

| Bước | Xử lý |
|---|---|
| 1 | Nhận ảnh từ ESP32-CAM |
| 2 | Lưu ảnh |
| 3 | Resize ảnh |
| 4 | Chuyển grayscale nếu cần |
| 5 | Denoise / blur |
| 6 | Chạy rule-based detection hoặc ML model |
| 7 | Trả JSON result |

### Response JSON mẫu

    {
      "request_id": "abc-123",
      "device_id": "esp32cam-01",
      "result": "person_detected",
      "confidence": 0.87
    }

---

## 6.5. Step 4 — Display AI Result on OLED

### OLED state map

| Server result | OLED text |
|---|---|
| idle | IDLE |
| motion_detected | MOTION |
| uploading | UPLOADING |
| processing | AI PROCESSING |
| person_detected | PERSON DETECTED |
| no_person | NO PERSON |
| error | SERVER ERROR |

---

## 6.6. Mini-project Phase 3.1 — Motion-triggered Cloud Vision

| Mục | Nội dung |
|---|---|
| Hardware | ESP32-S3 + PIR + OLED + ESP32-CAM |
| Trigger | PIR phát hiện chuyển động |
| Camera | ESP32-CAM chụp ảnh |
| Transport | HTTP POST qua OpenWrt VPN |
| Backend | FastAPI trên Proxmox |
| AI | Python/OpenCV hoặc model server-side |
| Feedback | OLED hiển thị kết quả |
| Success | Có motion → có ảnh → server xử lý → OLED hiện result |

---

## 6.7. Mini-project Phase 3.2 — AIoT Security Dashboard

| Mục | Nội dung |
|---|---|
| Hardware | ESP32-S3 + PIR + OLED + ESP32-CAM |
| Event stream | MQTT |
| Image upload | HTTP |
| Backend | FastAPI + MQTT subscriber |
| Storage | Local folder hoặc MinIO |
| Dashboard | Web dashboard / Node-RED / Grafana |
| Success | Dashboard hiển thị event, ảnh, kết quả AI |

---

# 7. Concise Hardware Pinout / Map

## 7.1. ESP32-S3 Node

| Component | Pin module | ESP32-S3 GPIO / Pin | Ghi chú |
|---|---|---|---|
| PIR HC-SR501 | VCC | 5V / VIN | Ưu tiên 5V |
| PIR HC-SR501 | GND | GND | Bắt buộc chung GND |
| PIR HC-SR501 | OUT | GPIO 4 | Digital input, thường logic 3.3V |
| OLED I2C | VCC | 3V3 | OLED nên dùng 3.3V |
| OLED I2C | GND | GND | Chung GND |
| OLED I2C | SDA | GPIO 8 | Có thể đổi trong code |
| OLED I2C | SCL | GPIO 9 | Có thể đổi trong code |

---

## 7.2. ESP32-CAM Node

| Component | Pin / Interface | Ghi chú |
|---|---|---|
| OV2640 camera | Built-in | Dùng thư viện esp32-camera trong Arduino Framework |
| USB programming shield | Micro-USB | Dùng để flash |
| Power | 5V ổn định | Ưu tiên USB mainboard sau case hoặc sạc 5V-2A |
| Flash LED | Built-in | Không bật ở bài test đầu tiên |
| Network | WiFi | Dễ gây spike dòng khi transmit |

---

# 8. Khi nào dùng MQTT và khi nào dùng HTTP?

| Nhu cầu | Giao thức nên dùng |
|---|---|
| PIR motion event | MQTT |
| Heartbeat | MQTT |
| Device status | MQTT |
| Server command | MQTT |
| Upload ảnh JPEG | HTTP |
| Gọi API AI | HTTP |
| Cần response trực tiếp | HTTP |
| Event realtime nhỏ | MQTT |

---

# 9. Milestones

## Milestone 1: Hardware Survival

Pass khi:

- Blink được ESP32-S3.
- Đọc được PIR ra Serial.
- OLED hiện Hello World.
- PIR + OLED hoạt động cùng nhau.
- Biết xử lý lỗi dây, nguồn, GPIO, Serial.
- Biết ESP32-CAM rất nhạy với nguồn.

---

## Milestone 2: IoT Connectivity

Pass khi:

- ESP32-S3 kết nối WiFi ổn định.
- Gửi JSON MQTT về Proxmox.
- Backend/MQTT subscriber nhận được event.
- OLED hiển thị trạng thái WiFi/MQTT.
- Kết nối chạy qua OpenWrt VPN.
- Firmware Phase 2 dùng `millis()` thay vì `delay()` dài.

---

## Milestone 3: AIoT Cloud Brain

Pass khi:

- ESP32-CAM chụp và upload ảnh.
- FastAPI nhận và lưu ảnh.
- Server xử lý ảnh bằng Python/OpenCV/AI.
- Server trả result.
- OLED hiển thị kết quả AI.

---

# 10. Final Survival Checklist

Khi lỗi, kiểm tra theo thứ tự:

| Thứ tự | Cần kiểm tra |
|---|---|
| 1 | Nguồn có đủ không? |
| 2 | GND đã nối chung chưa? |
| 3 | Cắm đúng GPIO chưa? |
| 4 | PIR đã cấp 5V / VIN chưa? |
| 5 | ESP32-CAM có bị brownout không? |
| 6 | Serial baud rate đúng chưa? |
| 7 | Có đang dùng `delay()` làm nghẽn MQTT không? |
| 8 | WiFi RSSI có yếu không? |
| 9 | OpenWrt firewall/VPN route có chặn không? |
| 10 | Broker/API server có đang chạy không? |
| 11 | Payload JSON có quá to không? |
| 12 | ESP32-CAM có đang bật flash LED làm sụt nguồn không? |

---

# 11. Final Strategy

Roadmap này ưu tiên sống sót và có kết quả nhanh:

| Không làm sớm | Làm trước |
|---|---|
| ESP-IDF | PlatformIO + Arduino Framework |
| Rust | C++ Arduino ổn định |
| TinyML on-device | AI trên Proxmox |
| Ghép nhiều linh kiện ngay | Atomic testing |
| Model ML trên ESP32 | HTTP upload ảnh về server |
| Kiến trúc phức tạp | End-to-end demo nhỏ nhưng chạy thật |
| `delay()` trong MQTT loop | `millis()` non-blocking |

Triết lý cuối cùng:

- ESP32 học cảm biến, camera, networking.
- OpenWrt bảo vệ đường truyền.
- Proxmox chạy backend và AI.
- OLED cho feedback tại chỗ.
- Nguồn cấp là thứ phải nghi ngờ đầu tiên khi ESP32-CAM lỗi.
- Từ Phase 2 trở đi, `millis()` là kỹ năng bắt buộc.
- Khi hệ thống này chạy ổn, lúc đó mới đáng học ESP-IDF, Rust hoặc TinyML.