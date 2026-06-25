# 🔷 Peak Project — Architecture Overview

> **Tài liệu**: Phân tích kiến trúc tổng thể của dự án Peak như một Senior Embedded Software Engineer.
>
> **Ngày**: 2026-06-25
>
> **Tác giả phân tích**: Sisyphus

---

## 1. Mục tiêu của dự án

**Peak** (tên firmware: **littleVisual**) là một **siêu mini smart terminal** (thiết bị đầu cuối thông minh) chạy trên vi điều khiển **ESP32-PICO-D4**, có khả năng hiển thị đồ họa màu qua màn hình **240×240 LCD** (ST7789) sử dụng thư viện **LVGL v8.1**.

Dự án là bản **port từ X-Track** (của FASTSHIFT, chạy trên AT32F403) sang nền tảng ESP32, được thực hiện bởi **稚晖 (PZH)** — mục đích tạo một thiết bị đeo tay thông minh có khả năng:

- Hiển thị thông số cảm biến (IMU 9-axis)
- Định vị GPS + bản đồ Offline
- Theo dõi lộ trình (Track recording)
- Nhạc (Audio/Buzzer)
- Kết nối Bluetooth
- Đang phát triển thêm: **3D engine** nhúng (software renderer dùng fixed-point math)

---

## 2. Các chức năng chính

| Chức năng | Mô tả |
|---|---|
| **Hiển thị GUI** | LVGL v8.1 trên màn hình 240×240 ST7789, 16-bit màu, 60 FPS |
| **Quản lý trang (Page Manager)** | Cơ chế Stack-based switching giống iOS ViewController, có animation |
| **Cảm biến IMU 6/9-axis** | MPU6050 hoặc MPU9250 — đo gia tốc, con quay, từ trường |
| **Định vị GPS** | Tọa độ + hiển thị bản đồ tile (Bing Maps) |
| **Theo dõi lộ trình** | Record, filter, và hiển thị track GPX |
| **Bluetooth** | NimBLE stack |
| **Audio** | Play nhạc nền qua loa/buzzer |
| **Quản lý nguồn** | Pin LiPo với chip sạc MCP73831, đo dung lượng, auto low-power |
| **Thẻ nhớ SD** | Lưu cấu hình, nhạc, dữ liệu GPS track |
| **3D Engine (đang phát triển)** | Software renderer dùng fixed-point arithmetic, mesh cube |

---

## 3. Kiến trúc tổng thể

Peak sử dụng kiến trúc **3 lớp (3-layer)** + **MVC** (Model-View-Controller) trong từng Page:

```
┌─────────────────────────────────────────────────────┐
│                     APP Layer                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │  Pages   │ │ Accounts │ │Resources │ │3D Engine│ │
│  │ (View)   │ │ (Model)  │ │(Font,Img)│ │(SW rndr)│ │
│  └────┬─────┘ └────┬─────┘ └──────────┘ └─────────┘ │
├───────┼─────────────┼────────────────────────────────┤
│       │   Framework Layer                            │
│  ┌────┴─────────────┴──────────────────────────────┐ │
│  │  AccountSystem (Pub/Sub Message Middleware)      │ │
│  │  PageManager (iOS ViewController-like stack)     │ │
│  │  LVGL Port (Display/Input/FS)                   │ │
│  │  Utilities: Filters, MapConv, TileConv,         │ │
│  │  GPX Parser, TrackFilter, TonePlayer,           │ │
│  │  ButtonEvent, StorageService, Time              │ │
│  └─────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────┤
│                   HAL Layer                           │
│  ┌──────┐ ┌──────┐ ┌────┐ ┌────┐ ┌────┐ ┌───────┐  │
│  │ IMU  │ │GPS*  │ │SD  │ │BT  │ │PWR │ │Buzzer │  │
│  │6050  │ │      │ │Card│ │Nim │ │    │ │/Audio │  │
│  └──────┘ └──────┘ └────┘ └────┘ └────┘ └───────┘  │
│  ┌──────┐ ┌──────────┐ ┌─────────┐                   │
│  │Encod │ │Backlight │ │I2C Scan │                   │
│  └──────┘ └──────────┘ └─────────┘                   │
├──────────────────────────────────────────────────────┤
│              Hardware (ESP32-PICO-D4)                 │
│  ST7789 240x240 | MPU6050 | SD Card | Buzzer         │
│  Rotary Encoder | LiPo Battery | Bluetooth           │
└──────────────────────────────────────────────────────┘
```

### Luồng khởi tạo (Boot Sequence)

```
main.setup()
  ├── HAL::Init()
  │     ├── malloc(lv_disp_buf)  ← cấp phát buffer màn hình trước
  │     ├── BT_Init()
  │     ├── Power_Init()
  │     ├── Backlight_Init()
  │     ├── Encoder_Init()
  │     ├── Buzz_init()
  │     ├── Audio_Init() + SD_Init() + I2C_Init() + IMU_Init()
  │     └── Audio_PlayMusic("Startup")
  ├── Port_Init()
  │     └── lv_port_disp_init(), lv_port_indev_init(), lv_fs_if_init()
  └── App_Init()
        ├── Accounts_Init() ← tạo tất cả Account node
        ├── Resource.Init() ← fonts, images
        ├── StatusBar::Init()
        ├── manager.Install("Template", "SystemInfos", "Startup")
        ├── manager.Push("Pages/Startup")
        └── Notify Storage + SysConfig LOAD
```

### Main Loop

```
main.loop()
  ├── HAL::Update()
  │     ├── Power_Update()
  │     ├── Encoder_Update()
  │     ├── Audio_Update()
  │     ├── IMU_Update()
  │     ├── BT_Update()
  │     └── SD_Update() (500ms interval)
  └── delay(20) ← ~50Hz loop rate
```

> LVGL chạy trên **RTOS task riêng** (`handleTaskLvgl`), được signal bởi `INIT_DONE()` sau khi App khởi tạo xong.

---

## 4. Các module chính

### 4.1 HAL Layer (`src/HAL/`)

Module phần cứng — abstract hóa toàn bộ ngoại vi ESP32:

| Module | File | Chức năng |
|---|---|---|
| **Backlight** | `HAL_Backlight.cpp` | Điều khiển độ sáng màn hình (PWM), gradual fade |
| **IMU** | `HAL_IMU.cpp` | MPU6050/9250: accel, gyro, magnetometer, attitude |
| **SD Card** | `HAL_SdCard.cpp` | SPI SD card, detect, event callback |
| **Power** | `HAL_Power.cpp` | Pin voltage, charging detect, shutdown, auto low-power |
| **Encoder** | `HAL_Encoder.cpp` | Rotary encoder A/B/push với event callback |
| **Buzzer** | `HAL_Buzz.cpp` | PWM buzzer/tone |
| **Audio** | `HAL_Audio.cpp` | Play WAV/audio từ SD card qua DAC |
| **Bluetooth** | `HAL_Bluetooth.cpp` | NimBLE BLE stack |
| **I2C** | `HAL_I2C_Scan.cpp` | I2C bus scan tool |

### 4.2 Framework Layer

#### AccountSystem (`src/App/Utils/AccountSystem/`)

**Message middleware** dạng Pub/Sub — xương sống giao tiếp giữa các module:

- **`Account`**: Một node (publisher/subscriber). Có thể `Commit()` (publish), `Subscribe()`, `Pull()`, `Notify()`, `SetTimerPeriod()`.
- **`AccountBroker`**: Trung tâm đăng ký — quản lý tất cả Account trong `AccountPool`, tìm kiếm theo ID.
- **`AccountMaster`**: Account đặc biệt tự động theo dõi tất cả — dùng cho broadcast/notify.

Các **Account node** được định nghĩa bởi `_ACT_LIST.inc`:

| Account | Cache | Chức năng |
|---|---|---|
| `Storage` | 0 | Lưu/tải cấu hình JSON lên SD |
| `Power` | 0 | Quản lý nguồn |
| `IMU` | size of IMU_Info_t | Dữ liệu cảm biến |
| `StatusBar` | 0 | Thanh trạng thái UI |
| `MusicPlayer` | 0 | Play nhạc |
| `SysConfig` | 0 | Cấu hình hệ thống |

Mỗi Account xử lý sự kiện thông qua callback `onEvent()` — switch case theo `EventCode_t`:

| Event | Ý nghĩa |
|---|---|
| `EVENT_PUB_PUBLISH` | Publisher đẩy dữ liệu |
| `EVENT_SUB_PULL` | Subscriber kéo dữ liệu |
| `EVENT_NOTIFY` | Notify 2 chiều |
| `EVENT_TIMER` | Timer định kỳ |

#### PageManager (`src/App/Utils/PageManager/`)

Quản lý vòng đời page theo cơ chế **stack** (Push/Pop) lấy cảm hứng từ **iOS ViewController**:

```
PageBase — vòng đời 8 trạng thái:

  IDLE → LOAD → WILL_APPEAR → DID_APPEAR → ACTIVITY
       → WILL_DISAPPEAR → DID_DISAPPEAR → UNLOAD
```

Các virtual method có thể override:

| Method | Thời điểm gọi |
|---|---|
| `onCustomAttrConfig()` | Cấu hình animation tùy chỉnh |
| `onViewLoad()` | Khởi tạo view (cấp phát controls) |
| `onViewDidLoad()` | Load hoàn tất |
| `onViewWillAppear()` | Trước khi hiện (thường dùng cho animation) |
| `onViewDidAppear()` | Đã hiện |
| `onViewWillDisappear()` | Chuẩn bị ẩn |
| `onViewDidDisappear()` | Đã ẩn |
| `onViewDidUnload()` | Hủy page, giải phóng tài nguyên |

Hỗ trợ animation chuyển trang: **Over** (đè), **Move** (đẩy), **Fade**, **None**.

**Các Page đã cài đặt:**

| Page | Đường dẫn | Chức năng |
|---|---|---|
| `StartUp` | `Pages/StartUp/` | Màn hình khởi động |
| `SystemInfos` | `Pages/SystemInfos/` | Hiển thị thông tin hệ thống |
| `StatusBar` | `Pages/StatusBar/` | Thanh trạng thái (top-layer riêng) |
| `Template` | `Pages/_Template/` | Template mẫu cho page mới |

Mỗi page tuân theo **MVC pattern**:

| Thành phần | File | Vai trò |
|---|---|---|
| **Controller** | `*Page.h/cpp` | Điều phối, xử lý sự kiện |
| **Model** | `*PageModel.h/cpp` | Dữ liệu, kết nối Account |
| **View** | `*PageView.h/cpp` | LVGL UI layout |

#### LVGL Port (`src/Port/`)

Port LVGL v8.1 lên ESP32:

| File | Chức năng |
|---|---|
| `lv_port_disp.cpp` | Display buffer (single buffer: 240×120 × 2 bytes) |
| `lv_port_indev.cpp` | Input device (encoder + nút nhấn) |
| `lv_port_fatfs.cpp` | File system driver cho SD card |

#### Utilities (`src/App/Utils/`)

| Module | Chức năng |
|---|---|
| **Filters** | Median, Lowpass, Hysteresis, Sliding filter cho dữ liệu cảm biến |
| **MapConv** | Chuyển đổi GPS ↔ tile coordinates (Bing Maps Tile System) |
| **TileConv** | Quản lý tile map |
| **GPX Parser** | Parse file GPX (GPS Exchange Format) |
| **TrackFilter** | Lọc điểm track (Douglas-Peucker) |
| **GPS_Transform** | Chuyển đổi WGS84 ↔ GCJ-02 |
| **TonePlayer** | Play melody qua buzzer |
| **ButtonEvent** | Xử lý event nút bấm với short/long press |
| **StorageService** | Ghi/đọc JSON config lên SD card |
| **lv_ext** | Extension cho LVGL: animation timeline, label animation |
| **lv_allocator** | Custom memory allocator cho LVGL heap |
| **Time** | Thư viện thời gian (TimeLib) |

### 4.3 APP Layer

#### 3D Engine (`src/App/Utils/3DEngine/`)

Software 3D renderer nhúng **fixed-point** (Q14.14) không dùng FPU:

- `Vector3` (long x, y, z), `Matrix4` identity với fixed-point (PRES = 16384)
- Phép nhân fixed-point: `((x * y) + PROUNDBIT) >> PSHIFT`
- Mesh cube: 8 đỉnh, 12 triangles (dạng index buffer)
- Dự kiến mở rộng cho hiển thị model 3D như Dummy示教器

#### Resources (`src/App/Resources/`)

| Loại | Nội dung |
|---|---|
| **Font** | AgencyB (36), Bahnschrift (13, 17, 32, 65) |
| **Image** | ~30 icon: battery, Bluetooth, compass, GPS arrow, menu, satellite, SD card, locate, trip, system info, v.v. (C array) |

---

## 5. Luồng dữ liệu giữa các module

```
                    ┌──────────────────┐
                    │   Hardware (I2C,  │
                    │   SPI, GPIO, BLE) │
                    └────────┬─────────┘
                             │ Raw data
                             ▼
                    ┌──────────────────┐
                    │   HAL Layer      │
                    │   (Drivers)      │
                    └────────┬─────────┘
                             │ Struct (IMU_Info_t, Power_Info_t...)
                             ▼
                    ┌─────────────────────┐
                    │  AccountSystem      │
                    │  (Pub/Sub Broker)   │
                    │                     │
                    │  Storage ← Power ← │ → StatusBar
                    │  SysConfig ← IMU ← │ → SystemInfos
                    │                    │ → MusicPlayer
                    └────────┬───────────┘
                             │ Account::Commit() / Pull()
                             ▼
                    ┌─────────────────────┐
                    │  Pages (MVC)        │
                    │  ┌─ Model ─┐        │
                    │  │ Account│◄────────│── Pull sensor data
                    │  └────────┘         │
                    │  ┌── View ──┐       │
                    │  │ LVGL GUI │       │
                    │  └──────────┘       │
                    └─────────────────────┘
```

### Ví dụ luồng IMU

1. **HAL::IMU_Update()** đọc I2C từ MPU6050 → cập nhật `IMU_Info_t`
2. **HAL** gọi `AccountSystem::IMU_Commit(&info)` → `Account("IMU").Commit(data)` vào PingPongBuffer
3. **Account System** gửi `EVENT_PUB_PUBLISH` đến subscribers
4. **Page SystemInfos** đã `Subscribe("IMU")` → `onEvent()` → `Pull()` dữ liệu → hiển thị lên LVGL

### Ví dụ luồng Encoder → Page Navigation

1. **HAL::Encoder_Update()** đọc xung encoder, tính diff
2. **HAL** gọi callback → **ButtonEvent** xử lý short/long press
3. **Account System** gửi event đến PageManager hoặc StatusBar
4. **Page** nhận event, thực hiện Push/Pop page hoặc scroll menu

---

## 6. Công nghệ được sử dụng

| Công nghệ | Mục đích |
|---|---|
| **ESP32-PICO-D4** | MCU: Xtensa dual-core 240MHz, 520KB SRAM, 4MB Flash |
| **PlatformIO** | Build system & toolchain |
| **Arduino Framework** | ESP32 Arduino Core |
| **FreeRTOS** | RTOS (qua Arduino ESP32) |
| **LVGL v8.1** | Embedded GUI library (widgets, animation, styles) |
| **TFT_eSPI** | SPI LCD driver (tối ưu cho ESP32) |
| **NimBLE** | Bluetooth Low Energy stack |
| **FatFS** | File system trên SD card |
| **ArduinoJson** | JSON parser/serializer cho config |
| **MPU6050/9250** | 6/9-axis IMU sensor |
| **ST7789** | 240×240 TFT LCD controller |

---

## 7. Framework được sử dụng

| Framework | Vai trò | Loại |
|---|---|---|
| **Arduino ESP32** | Platform SDK, WiFi/BT stack, RTOS wrapper | Bên thứ ba |
| **LVGL v8.1** | Graphics framework — rendering, input, animations | Bên thứ ba |
| **AccountSystem** | Pub/Sub message middleware nội bộ | Tự viết |
| **PageManager** | Page lifecycle quản lý theo stack | Tự viết |
| **TFT_eSPI** | Low-level LCD driver (SPI + DMA) | Bên thứ ba |
| **NimBLE-Arduino** | BLE host stack | Bên thứ ba |

---

## 8. Sơ đồ kiến trúc dạng text

```
PEAK (littleVisual) - Embedded Smart Terminal
============================================

┌──────────────────────────────────────────────────────────────────┐
│   main.cpp                                                       │
│   setup(): HAL::Init() → Port_Init() → App_Init()              │
│   loop():  HAL::Update() → delay(20)                           │
└────────────────────────┬─────────────────────────────────────────┘
                         │
┌────────────────────────┴─────────────────────────────────────────┐
│  HAL Layer  (src/HAL/)                                           │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────────┐ │
│  │ IMU  │  SD  │ PWR  │ Enc  │Buzz  │Audio │  BT  │Backlight  │ │
│  │6050  │ Card │ Mgmt │oder  │      │Player│ NIM  │           │ │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴───────────┘ │
├──────────────────────────────────────────────────────────────────┤
│  Framework Layer                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  AccountSystem (Pub/Sub Message Bus)                        │ │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐  │ │
│  │  │ Storage   │ │ Power     │ │ IMU       │ │ SysConfig │  │ │
│  │  │ Account   │ │ Account   │ │ Account   │ │ Account   │  │ │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  PageManager (Stack-based Page Lifecycle)                   │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                    │ │
│  │  │ StartUp  │ │SystemInf │ │StatusBar │ ← Pages on stack   │ │
│  │  │ Page     │ │os Page   │ │ (top)    │                    │ │
│  │  └──────────┘ └──────────┘ └──────────┘                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  LVGL Port                  │  Utilities                    │ │
│  │  ├─ lv_port_disp (display)  │  ├─ Filters (Median,LPF,...) │ │
│  │  ├─ lv_port_indev (encoder) │  ├─ MapConv / TileConv       │ │
│  │  └─ lv_port_fatfs (SD I/O) │  ├─ GPX Parser / TrackFilter │ │
│  │                             │  ├─ TonePlayer / ButtonEvent │ │
│  │                             │  └─ StorageService / Time    │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│  APP Layer                                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Pages (MVC Pattern)        │  Resources                   │ │
│  │  ├─ StartUp:  Model+View+Ctrl│  ├─ Font: AgencyB,          │ │
│  │  ├─ SystemInfos: MVC        │  │   Bahnschrift x5         │ │
│  │  ├─ StatusBar (top overlay) │  ├─ Image: ~30 icons (C)    │ │
│  │  └─ Template (boilerplate)  │  └─ 3D Engine (fixed-point) │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│  Hardware Platform (ESP32-PICO-D4)                               │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌───────┐ ┌──────────┐  │
│  │ST7789   │ │MPU6050  │ │SD Card   │ │Encod. │ │Battery   │  │
│  │240x240  │ │I2C(0x68)│ │SPI       │ │Rotary │ │LiPo+Chgr │  │
│  └─────────┘ └─────────┘ └──────────┘ └───────┘ └──────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Dependency giữa các module

```
main.cpp
  ├── HAL/
  │     ├── HAL.h → HAL_Def.h, Config.h, CommonMacro.h, FreeRTOS
  │     ├── HAL_IMU.cpp → Wire (I2C)
  │     ├── HAL_SdCard.cpp → SD, SPI
  │     ├── HAL_Power.cpp → ADC, GPIO
  │     ├── HAL_Encoder.cpp → GPIO interrupt
  │     ├── HAL_Bluetooth.cpp → NimBLE library
  │     ├── HAL_Audio.cpp → I2S / DAC
  │     └── HAL_Buzz.cpp → LEDC (PWM)
  │
  ├── Port/
  │     ├── Display.h → Config.h, LVGL, TFT_eSPI
  │     ├── Display.cpp → TFT_eSPI, lv_port_*
  │     ├── lv_port_disp.cpp → LVGL disp driver, malloc buffer
  │     ├── lv_port_indev.cpp → LVGL input, Encoder, Button
  │     └── lv_port_fatfs.cpp → LVGL FS driver, FatFS
  │
  └── App/
        ├── App.h → App.cpp → Account_Master, PageManager, ResourcePool
        │     ├── Account_Master.cpp → AccountBroker + _ACT_LIST.inc
        │     │     ├── ACT_STORAGE.cpp → Account, StorageService, ArduinoJson
        │     │     ├── ACT_POWER.cpp → Account, HAL_Power
        │     │     ├── ACT_IMU.cpp → Account, HAL_IMU
        │     │     ├── ACT_MusicPlayer.cpp → Account, TonePlayer
        │     │     └── ACT_SysConfig.cpp → Account, Storage (pull)
        │     │
        │     ├── PageManager → PageBase (LVGL obj)
        │     │     ├── Pages/StartUp/ → PageBase, Account (Storage)
        │     │     ├── Pages/SystemInfos/ → PageBase, Account (IMU, Power, Storage)
        │     │     ├── Pages/StatusBar/ → LVGL lv_layer_top()
        │     │     └── Pages/_Template/ → boilerplate
        │     │
        │     ├── ResourcePool → Font/Image C arrays
        │     └── Utils/
        │           ├── 3DEngine/ → math.h (fixed-point)
        │           ├── Filters/{Median,Lowpass,Hysteresis,Sliding}.h
        │           ├── MapConv/ → GPS_Transform, TileSystem
        │           ├── TrackFilter → math.h
        │           ├── TonePlayer → HAL_Buzz
        │           ├── ButtonEvent → GPIO
        │           └── StorageService → SD card, ArduinoJson
        │
        └── Configs/
              ├── Config.h → Pin definitions, system params
              └── Version.h → Firmware version info
```

### Đồ thị phụ thuộc mức cao

```
HAL ◄────── AccountSystem ◄────── Pages
  │                │                  │
  │         ┌──────┘                  │
  ▼         ▼                         ▼
Drivers   LVGL Port            LVGL Rendering
  │         │                        │
  └─────────┴────────────────────────┘
                    │
                    ▼
              ESP32 Hardware
         (GPIO, I2C, SPI, I2S, BLE)
```

---

## Phụ lục: Cấu trúc thư mục dự án Peak

```
Peak/
├── 1.Hardware/                  ← Thiết kế hardware (Altium Designer)
│   └── Peak/
│       ├── Main.SchDoc          ← Schematic
│       ├── Peak.PcbDoc          ← PCB Layout
│       └── Peak.PrjPCB          ← Altium Project
│
├── 2.Firmware/                  ← Firmware ESP32
│   ├── Libraries/               ← Thư viện ngoài (SD, SPI, TFT_eSPI, NimBLE)
│   └── PlatformIO/
│       └── Peak-ESP32-fw/
│           ├── platformio.ini   ← PlatformIO config (pico32, Arduino)
│           ├── src/
│           │   ├── main.cpp     ← Entry point
│           │   ├── HAL/         ← Hardware Abstraction Layer
│           │   ├── Port/        ← LVGL port (display, input, filesystem)
│           │   └── App/         ← Application layer
│           │       ├── App.h/cpp
│           │       ├── Configs/ (Config.h, Version.h)
│           │       ├── Accounts/ (data processing nodes)
│           │       ├── Pages/ (StartUp, SystemInfos, StatusBar, Template)
│           │       ├── Resources/ (Font, Image)
│           │       └── Utils/ (AccountSystem, PageManager, Filters, ...)
│           └── test/
│
├── 3.Software/                  ← PC Tools
│   ├── ImageConvertor/         ← Python tool: convert ảnh → C array
│   │   ├── main.py
│   │   ├── converter.py
│   │   └── filename_renamer.py
│   └── Simulator/              ← Windows LVGL Simulator (Visual Studio)
│       └── LVGL.Simulator/
│           ├── LVGL.Simulator.sln
│           ├── HAL/ (Simulator HAL implementations)
│           ├── lv_drivers/ (SDL/GTK drivers)
│           └── lib/ (Stream, WString cho Arduino compat)
│
├── 4.Model/                     ← 3D case CAD (STEP files)
│   ├── 主件.step               ← Main body
│   └── 侧件.step               ← Side part
│
├── 5.Docs/                      ← Tài liệu
│   ├── 1.Datasheets/           ← Datasheets (ESP32, ST7789, MPU9250, v.v.)
│   ├── 2.Images/               ← Hình ảnh kiến trúc, sơ đồ
│   └── 3.References/           ← Tài liệu tham khảo
│
└── README.md                    ← Tổng quan dự án
```

---

## Tổng kết

Peak là một **embedded project có cấu trúc rất tốt** (disciplined codebase). Những điểm nổi bật:

1. **Clean Architecture**: Phân tách rõ ràng 3 lớp HAL → Framework → APP
2. **MVC Pattern**: Mỗi Page tách biệt Model, View, Controller
3. **Pub/Sub Messaging**: AccountSystem giúp tách rời data producers (HAL) và consumers (UI)
4. **State-machine Lifecycle**: PageManager với 8 trạng thái quản lý page chặt chẽ
5. **Fixed-point 3D Engine**: Software renderer nhúng không cần FPU
6. **Porting Optimization**: Nhiều kỹ thuật tối ưu memory (malloc sớm, PingPong buffer, LV_MEM_SIZE) và SPI timing (IOMUX, HSPI/VSPI)
7. **Cross-platform**: Source code tương thích cả ESP32 (Arduino) và Windows Simulator (Visual Studio)
