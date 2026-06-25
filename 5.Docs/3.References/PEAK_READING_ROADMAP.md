# 🔷 Peak — Source Code Reading Roadmap

> Lộ trình đọc hiểu toàn bộ firmware Peak dành cho người mới.
> Đi từ tổng quan → chi tiết → phần cứng.
> Mỗi phase có file cần đọc, thứ tự, và mục tiêu cụ thể.

---

## Trước khi bắt đầu

### Công cụ cần có

| Công cụ | Mục đích |
|---|---|
| VS Code + PlatformIO | Mở và duyệt project |
| `find` / `grep` | Tìm kiếm trong codebase |
| `git log --oneline` | Xem lịch sử commit |
| `cloc src/` | Đếm số dòng code |

### Cấu trúc thư mục tổng quan

```
Peak/
├── 1.Hardware/       ← Thiết kế PCB (Altium Designer)
├── 2.Firmware/       ← Firmware ESP32 (code chính)
│   └── PlatformIO/Peak-ESP32-fw/
│       ├── platformio.ini
│       ├── src/          ← Source code
│       │   ├── main.cpp
│       │   ├── HAL/      ← Hardware Abstraction
│       │   ├── Port/     ← LVGL + Filesystem port
│       │   └── App/      ← Application
│       ├── lib/          ← Third-party libs
│       └── test/
├── 3.Software/       ← PC Tools (ImageConvertor, Simulator)
├── 4.Model/          ← 3D case (STEP files)
└── 5.Docs/           ← Tài liệu
```

> **Tổng quan**: Project ~8.000 dòng code C/C++ (không tính thư viện).
>
> | Layer | Files | Dòng code (ước lượng) |
> |---|---|---|
> | HAL | 10 | ~500 |
> | Port | 4 | ~550 |
> | Framework | 15 | ~2,500 |
> | App (Accounts + Pages + Resources) | 25 | ~2,000 |
> | Config | 3 | ~160 |
> | **Total** | **~57** | **~5,800** |

---

## Phase 1: Hiểu kiến trúc

> ⏱ Thời gian: 30–45 phút
>
> 🎯 Mục tiêu: Biết được dự án làm gì, các lớp kiến trúc, và luồng boot.

### Thứ tự đọc

```
Step 1 ── README.md                   (toàn cảnh dự án)
Step 2 ── platformio.ini              (cấu hình build)
Step 3 ── src/                        (cấu trúc thư mục)
Step 4 ── main.cpp                    (entry point)
Step 5 ── HAL/HAL.cpp                 (HAL Init/Update)
Step 6 ── Port/Display.cpp            (Port Init)
Step 7 ── App/App.cpp                 (App Init)
Step 8 ── App/Configs/Config.h        (cấu hình toàn bộ)
Step 9 ── App/Configs/Version.h       (version info)
```

### File cần đọc

| Step | File | Dòng | Nội dung chính |
|---|---|---|---|
| 1 | `README.md` | 2 | Dòng mô tả đầu tiên: "modular embedded application framework..." |
| 2 | `platformio.ini` | 22 | Board pico32, framework arduino, upload speed, SPIFFS |
| 3 | `src/` | — | 5 thư mục chính: main.cpp, HAL, Port, App |
| 4 | `src/main.cpp` | 18 | `setup()` gọi 3 hàm: `HAL::Init` → `Port_Init` → `App_Init` |
| 5 | `src/HAL/HAL.cpp` | 42 | Khởi tạo 9 module phần cứng theo thứ tự |
| 6 | `src/Port/Display.cpp` | 71 | Màn hình + LVGL + Tạo LvglThread (FreeRTOS) |
| 7 | `src/App/App.cpp` | 60 | Accounts, Resources, Pages, StatusBar, INIT_DONE |
| 8 | `src/App/Configs/Config.h` | 92 | Pin map, kích thước màn hình, GPS default, đường dẫn |
| 9 | `src/App/Configs/Version.h` | 80 | Firmware name "littleVisual", version, compiler info |

### Mục tiêu cần đạt

Sau Phase 1, bạn trả lời được:

- [x] Dự án làm gì? → Smart terminal ESP32, port từ X-Track
- [x] Ai viết? → 稚晖 (PZH)
- [x] Chip gì? → ESP32-PICO-D4
- [x] Kiến trúc mấy lớp? → 3 lớp: HAL → Framework → APP
- [x] Luồng boot thế nào? → `setup()` → HAL::Init → Port_Init → App_Init → `loop()` gọi HAL::Update
- [x] Build bằng gì? → PlatformIO, board pico32, Arduino framework
- [x] Màn hình gì? → 240×240 ST7789, LVGL v8.1
- [x] File nào là chính? → main.cpp, HAL.cpp, Display.cpp, App.cpp

---

## Phase 2: Hiểu GUI

> ⏱ Thời gian: 1–2 giờ
>
> 🎯 Mục tiêu: Hiểu cách LVGL hoạt động, PageManager quản lý màn hình, MVC pattern, navigation.

### Thứ tự đọc

```
Step 1 ── App/Resources/ResourcePool.h/.cpp    (fonts + images)
Step 2 ── App/Pages/AppFactory.h/.cpp          (page factory)
Step 3 ── Utils/PageManager/PageBase.h         (base class)
Step 4 ── Utils/PageManager/PageManager.h      (manager API)
Step 5 ── Utils/PageManager/PM_State.cpp       (state machine)
Step 6 ── Utils/PageManager/PM_Router.cpp      (push/pop)
Step 7 ── Utils/PageManager/PM_Base.cpp        (install/register)
Step 8 ── App/Pages/StartUp/StartUp.h/.cpp     (page 1)
Step 9 ── App/Pages/StartUp/StartUpView.cpp    (view)
Step 10 ─ App/Pages/StartUp/StartUpModel.cpp   (model)
Step 11 ─ App/Pages/_Template/Template.h/.cpp  (page 2)
Step 12 ─ App/Pages/SystemInfos/SystemInfos.cpp (page 3)
Step 13 ─ App/Pages/StatusBar/StatusBar.cpp    (overlay)
Step 14 ─ App/Pages/Page.h                     (convenience header)
Step 15 ─ Port/lv_port/lv_port_disp.cpp        (LVGL display port)
Step 16 ─ Port/lv_port/lv_port_indev.cpp       (LVGL input port)
```

### File cần đọc

| Step | File | Dòng | Nội dung chính |
|---|---|---|---|
| 1 | `ResourcePool.h/.cpp` | 97 | Font (5): bahnschrift_13/17/32/65, agencyb_36. Image (27): battery, BT, compass, GPS... |
| 2 | `AppFactory.h/.cpp` | 52 | `CreatePage(name)` tạo instance: Template, SystemInfos, Startup |
| 3 | `PageBase.h` | 164 | Lớp base: 8 trạng thái (IDLE→LOAD→...→UNLOAD), root LVGL object, stash, cache |
| 4 | `PageManager.h` | 215 | API: Push/Pop/Install/BackHome, Animation types (OVER/MOVE/FADE/NONE) |
| 5 | `PM_State.cpp` | 230 | State machine: LOAD→WILL_APPEAR→DID_APPEAR→ACTIVITY→WILL_DISAPPEAR→... |
| 6 | `PM_Router.cpp` | 497 | `Push()`: tìm page→push stack→SwitchTo. `SwitchTo()`: cập nhật state animation |
| 7 | `PM_Base.cpp` | 312 | `Install()`: Factory→CreatePage→onCustomAttrConfig→Register vào PagePool |
| 8 | `StartUp.h/.cpp` | 66 | Controller: `onViewLoad` (timer 2s), `onViewDidDisappear` (StatusBar appear) |
| 9 | `StartUpView.cpp` | 52 | View: container 110×40, label "DUMMY", animation timeline (width 0→110, y slide) |
| 10 | `StartUpModel.cpp` | 24 | Model: tạo Account "StartupModel", Notify MusicPlayer |
| 11 | `Template.h/.cpp` | 100 | Page mẫu: canvas vẽ, group focus, nhấn→Push SystemInfos. OVER_BOTTOM bounce |
| 12 | `SystemInfos.cpp` | 141 | 6 items (flex column), 100ms timer pull data, nhấn→Push Scene3D |
| 13 | `StatusBar.cpp` | 336 | Top layer overlay: battery SD BT icons, 1000ms update timer, slide appear animation |
| 14 | `Page.h` | 35 | Include convenience: PageManager + lv_ext + ResourcePool + StatusBar |
| 15 | `lv_port_disp.cpp` | 74 | Display flush callback, buffer 240×120, register LVGL display driver |
| 16 | `lv_port_indev.cpp` | 94 | Input device: ENCODER type, `encoder_read()` gọi HAL::Encoder_GetDiff() |

### Mục tiêu cần đạt

Sau Phase 2, bạn trả lời được:

- [x] LVGL version? → v8.1
- [x] Có mấy page? → 3 + status bar
- [x] Mỗi page gồm mấy phần? → 3: Controller (.cpp) + View (View.cpp) + Model (Model.cpp)
- [x] Navigation kiểu gì? → Stack (Push/Pop, giống iOS UINavigationController)
- [x] Animation? → OVER_TOP 500ms (global), OVER_BOTTOM bounce (Template), NONE (StartUp)
- [x] StatusBar ở đâu? → lv_layer_top(), ngoài page stack
- [x] Có mấy fonts? → 5 bahnschrift + 1 agencyb
- [x] Có mấy icons? → 27
- [x] Input device? → 1 encoder (LV_INDEV_TYPE_ENCODER)
- [x] Display buffer? → 240×120 (half screen), single buffer

---

## Phase 3: Hiểu RTOS

> ⏱ Thời gian: 45–60 phút
>
> 🎯 Mục tiêu: Hiểu các FreeRTOS task, priority, cơ chế đồng bộ, ISR, và giao tiếp giữa các task.

### Thứ tự đọc

```
Step 1 ── Port/Display.cpp                  (LvglThread)
Step 2 ── Port/Display.h                    (handleTaskLvgl)
Step 3 ── App/App.h                         (INIT_DONE macro)
Step 4 ── HAL/HAL_Buzz.cpp                  (BuzzerThread)
Step 5 ── HAL/HAL_Encoder.cpp               (GPIO ISR)
Step 6 ── HAL/HAL_Power.cpp                 (blocking delay 1s)
Step 7 ── HAL/HAL_Bluetooth.cpp             (NimBLE init)
Step 8 ── HAL/HAL.h                         (FreeRTOS.h include)
Step 9 ── HAL/CommonMacro.h                 (macro helpers)
```

### File cần đọc

| Step | File | Dòng | Nội dung chính |
|---|---|---|---|
| 1 | `Port/Display.cpp` | 71 | **Task LVGL**: `xTaskCreate(TaskLvglUpdate, "LvglThread", 20000, ..., MAX-1, &handleTaskLvgl)` — blocking `ulTaskNotifyTake()` |
| 2 | `Port/Display.h` | 41 | `extern TaskHandle_t handleTaskLvgl` |
| 3 | `App/App.h` | 51 | `INIT_DONE()` = `xTaskNotifyGive(handleTaskLvgl)` — đánh thức LVGL |
| 4 | `HAL/HAL_Buzz.cpp` | 60 | **Task Buzzer**: `xTaskCreate(BuzzerThread, "BuzzerThread", 800, ..., 1, &handleBuzzerThread)` — priority 1, 800 words stack |
| 5 | `HAL/HAL_Encoder.cpp` | 118 | **ISR**: `attachInterrupt(CONFIG_ENCODER_A_PIN, Encoder_IrqHandler, CHANGE)` — volatile EncoderDiff |
| 6 | `HAL/HAL_Power.cpp` | 167 | **Blocking delay 1s**: `while(millis()-time < 1000) { BT_Update(); delay(100); }` — giữ nguồn chờ ổn định |
| 7 | `HAL/HAL_Bluetooth.cpp` | 284 | **NimBLE**: `NimBLEDevice::init("Peak-BLE")` → tạo ble host + ll tasks |
| 8 | `HAL/HAL.h` | 77 | Include `FreeRTOS.h` — mang FreeRTOS API vào toàn bộ HAL |
| 9 | `CommonMacro.h` | 169 | Macro: `__IntervalExecute(func,time)` → static lasttime + millis() check |

### Task Map (tóm tắt)

```
┌──────────────┬──────────┬──────────┬──────────┬──────────────┐
│ Task         │ Priority │ Stack    │ Created  │ Chức năng     │
├──────────────┼──────────┼──────────┼──────────┼──────────────┤
│ LvglThread   │ MAX-1    │ 20000w   │ Port     │ LVGL render   │
│ loopTask     │ 1        │ default  │ Arduino  │ setup + loop  │
│ BuzzerThread │ 1        │ 800w     │ HAL_Buzz │ Phát tone     │
│ ble (NimBLE) │ MAX-4    │ NIMBLE   │ NimBLE   │ BLE host      │
│ ll (NimBLE)  │ MAX-1    │ MIN+400  │ NimBLE   │ BLE link layer│
└──────────────┴──────────┴──────────┴──────────┴──────────────┘
```

### Mục tiêu cần đạt

Sau Phase 3, bạn trả lời được:

- [x] Có mấy user task? → 3: loopTask, LvglThread, BuzzerThread
- [x] Task nào quan trọng nhất? → LvglThread (priority MAX-1, stack 80KB)
- [x] LvglThread bắt đầu chạy khi nào? → Sau `INIT_DONE()` signal
- [x] Cơ chế đồng bộ chính? → `xTaskNotifyGive` / `ulTaskNotifyTake`
- [x] Có queue/semaphore/mutex/event group không? → **Không** (tự tạo)
- [x] Có ISR không? → 1: Encoder GPIO interrupt
- [x] Giao tiếp task? → Shared volatile (EncoderDiff), AccountSystem (cùng task)
- [x] Stack lớn nhất? → LvglThread 20000 words (~80KB)
- [x] Priority cao nhất? → MAX-1 (LvglThread + NimBLE LL)
- [x] Có những FreeRTOS API nào được dùng? → `xTaskCreate`, `xTaskNotifyGive`, `ulTaskNotifyTake`, `vTaskDelay` (delay), `configMAX_PRIORITIES`

---

## Phase 4: Hiểu Message Framework

> ⏱ Thời gian: 1.5–2 giờ
>
> 🎯 Mục tiêu: Hiểu AccountSystem — Pub/Sub message middleware. Đây là xương sống của toàn bộ giao tiếp dữ liệu.

### Thứ tự đọc

```
Step 1 ── Utils/AccountSystem/Account.h              (Account class)
Step 2 ── Utils/AccountSystem/AccountBroker.h        (Broker class)
Step 3 ── Utils/AccountSystem/PingPongBuffer/PingPongBuffer.h (double buffer)
Step 4 ── Utils/AccountSystem/Account.cpp            (implementation)
Step 5 ── Utils/AccountSystem/AccountBroker.cpp      (implementation)
Step 6 ── Utils/AccountSystem/PingPongBuffer/PingPongBuffer.c
Step 7 ── App/Accounts/_ACT_LIST.inc                 (danh sách node)
Step 8 ── App/Accounts/ACT_Def.h                     (message structs)
Step 9 ── App/Accounts/Account_Master.h/.cpp         (khởi tạo)
Step 10 ─ App/Accounts/ACT_IMU.cpp                   (ví dụ publisher)
Step 11 ─ App/Accounts/ACT_Power.cpp                 (ví dụ pull + timer)
Step 12 ─ App/Accounts/ACT_Storage.cpp               (ví dụ notify)
Step 13 ─ App/Accounts/ACT_MusicPlayer.cpp           (ví dụ pure event)
Step 14 ─ App/Accounts/ACT_SysConfig.cpp             (ví dụ config)
Step 15 ─ App/Pages/SystemInfos/SystemInfosModel.cpp (subscriber)
Step 16 ─ App/App.h                                  (ACCOUNT_SEND_NOTIFY_CMD)
```

### File cần đọc

| Step | File | Dòng | Nội dung chính |
|---|---|---|---|
| 1 | `Account.h` | 118 | Class Account: ID, eventCallback, timer, BufferManager, publishers[], subscribers[]. 4 EventCode_t |
| 2 | `AccountBroker.h` | 53 | AccountBroker: AccountPool, AccountMaster, SearchAccount |
| 3 | `PingPongBuffer.h` | 51 | Double buffer: buffer[2], readIndex, writeIndex, readAvaliable[2] |
| 4 | `Account.cpp` | 500 | **Quan trọng nhất**: `Commit()`, `Publish()`, `Pull()` (2 overloads), `Notify()` (2), `Subscribe()`, `Unsubscribe()`, `SetTimerPeriod()` |
| 5 | `AccountBroker.cpp` | 156 | `AddAccount()` → thêm vào pool + AccountMaster.Subscribe(id), `SearchAccount()` |
| 6 | `PingPongBuffer.c` | 98 | `GetWriteBuf()`, `SetWriteDone()`, `GetReadBuf()`, `SetReadDone()` |
| 7 | `_ACT_LIST.inc` | 8 | 6 nodes: Storage(0), Power(0), IMU(36), StatusBar(0), MusicPlayer(0), SysConfig(0) |
| 8 | `ACT_Def.h` | 126 | Struct: `IMU_Info_t`, `Power_Info_t`, `Storage_Info_t`, `SysConfig_Info_t`, `MusicPlayer_Info_t`, `StatusBar_Info_t`, macro `STORAGE_VALUE_REG` |
| 9 | `Account_Master.h/.cpp` | 29 | `Accounts_Init()`: X-Macro đọc `_ACT_LIST.inc` → tạo Account + gọi Init. `AccountSystem::Broker()` |
| 10 | `ACT_IMU.cpp` | 14 | **Publisher đơn giản nhất**: lưu global ptr, chỉ `Commit()` từ HAL |
| 11 | `ACT_Power.cpp` | 60 | **Pull + Timer**: `EVENT_TIMER` check sạc, `EVENT_SUB_PULL` trả filtered data |
| 12 | `ACT_Storage.cpp` | 156 | **Notify**: load/save config, pull SysConfig, quản lý MapConv. Subscribe("SysConfig") |
| 13 | `ACT_MusicPlayer.cpp` | 26 | **Pure event**: notify → `HAL::Audio_PlayMusic()` |
| 14 | `ACT_SysConfig.cpp` | 62 | **Config**: 6 field đăng ký với Storage, `SUB_PULL` trả config |
| 15 | `SystemInfosModel.cpp` | 90 | **Subscriber**: Subscribe("IMU", "Power", "Storage"), Pull dữ liệu |
| 16 | `App.h` | 51 | Macro `ACCOUNT_SEND_NOTIFY_CMD(Storage,STORAGE_CMD_LOAD)` → broadcast command |

### Sơ đồ kết nối (tóm tắt)

```
Publisher              Subscriber
─────────────────────  ─────────────────────────
IMU (buffer=36)        → SystemInfosModel (Pull)
Power (buffer=0)       → StatusBar (Pull)
                       → SystemInfosModel (Pull)
                       → MusicPlayer (Notify) ← khi sạc
Storage (buffer=0)     → StatusBar (Pull)
                       → SystemInfosModel (Pull)
                       → SysConfig (Subscribe) ← pull config (để lưu)
MusicPlayer (buffer=0) → (chỉ nhận Notify)
SysConfig (buffer=0)   → Storage (Subscribe) ← đăng ký value để lưu
```

### Mục tiêu cần đạt

Sau Phase 4, bạn trả lời được:

- [x] Message framework tên gì? → AccountSystem
- [x] Kiểu Pub/Sub gì? → Đồng bộ (callback trực tiếp, không queue)
- [x] Có thread-safe không? → **Không** — tất cả trong 1 task
- [x] Event types? → 4: PUBLISH, PULL, NOTIFY, TIMER
- [x] Cơ chế cache? → PingPongBuffer (double buffer, 2 slots)
- [x] Có mấy Account node? → 6 (+1 AccountMaster)
- [x] AccountMaster làm gì? → Auto-subscribe tất cả, dùng broadcast
- [x] IMU dùng cơ chế nào? → Commit + Publish (push)
- [x] Power dùng cơ chế nào? → Pull (subscriber kéo) + Timer (kiểm tra sạc)
- [x] Storage dùng cơ chế nào? → Notify (nhận lệnh) + Pull (trả SD info)
- [x] Page Model dùng gì? → Subscribe + Pull
- [x] Làm sao gửi lệnh broadcast? → `ACCOUNT_SEND_NOTIFY_CMD` + AccountMaster
- [x] Các loại message struct? → IMU_Info_t (36B), Power_Info_t (4B), Storage_Info_t, SysConfig_Info_t, MusicPlayer_Info_t, StatusBar_Info_t
- [x] 3 bước của Push model? → Commit (ghi buffer) → Publish (gửi event) → callback subscriber

---

## Phase 5: Hiểu Driver

> ⏱ Thời gian: 2–3 giờ
>
> 🎯 Mục tiêu: Hiểu chi tiết từng driver HAL: cách giao tiếp với hardware, cấu hình, interrupt.

### Thứ tự đọc

```
Step 1  ─ HAL/HAL.h              (API toàn bộ HAL)
Step 2  ─ HAL/HAL_Def.h          (struct dữ liệu)
Step 3  ─ HAL/HAL.cpp            (Init/Update coordinator)
Step 4  ─ HAL/HAL_IMU.cpp        (I2C IMU sensor)
Step 5  ─ HAL/HAL_Encoder.cpp    (GPIO interrupt encoder)
Step 6  ─ HAL/HAL_Backlight.cpp  (PWM backlight + LVGL anim)
Step 7  ─ HAL/HAL_Power.cpp      (ADC battery + power sequencing)
Step 8  ─ HAL/HAL_Buzz.cpp       (PWM buzzer + RTOS thread)
Step 9  ─ HAL/HAL_Audio.cpp      (TonePlayer meldoy)
Step 10 ─ HAL/HAL_SdCard.cpp     (SPI SD card)
Step 11 ─ HAL/HAL_I2C_Scan.cpp   (I2C bus scan)
Step 12 ─ HAL/HAL_Bluetooth.cpp  (NimBLE BLE)
Step 13 ─ Port/lv_port/lv_port_disp.cpp    (SPI LCD flush)
Step 14 ─ Port/lv_port/lv_port_fatfs.cpp   (FatFS SD)
Step 15 ─ Utils/ButtonEvent/ButtonEvent.h  (button driver)
Step 16 ─ Utils/TonePlayer/TonePlayer.h    (melody player)
Step 17 ─ App/Utils/StorageService/StorageService.h (JSON config)
Step 18 ─ App/Utils/Filters/HysteresisFilter.h (filter mẫu)
Step 19 ─ App/Utils/Filters/MedianQueueFilter.h
```

### File cần đọc

| Step | File | Dòng | Nội dung chính |
|---|---|---|---|
| 1 | `HAL.h` | 77 | API: Init, Update + 9 module APIs (Backlight, IMU, SD, Power, Encoder, Buzz, Audio, BT, I2C) |
| 2 | `HAL_Def.h` | 49 | Struct: `Clock_Info_t`, `IMU_Info_t` (ax,ay,az,gx,gy,gz,mx,my,mz,roll,pitch,yaw), `Power_Info_t` (voltage,usage,isCharging) |
| 3 | `HAL.cpp` | 42 | **Init sequence**: malloc buffer → BT → Power (1s delay) → Backlight → Encoder → Buzz → Audio → SD → I2C → IMU → "Startup" sound |
| 4 | `HAL_IMU.cpp` | 34 | MPU9250 trên I2C 0x68. `IMU_Update()`: đọc 9-axis + attitude → `AccountSystem::IMU_Commit()` |
| 5 | `HAL_Encoder.cpp` | 118 | GPIO interrupt (CHANGE) trên encoder A. Hệ thống: ISR ghi EncoderDiff → `lv_port_indev` đọc → LVGL input |
| 6 | `HAL_Backlight.cpp` | 78 | **LEDC**: PWM 5kHz, 10-bit. `Backlight_SetGradual()` dùng `lv_anim` để fade |
| 7 | `HAL_Power.cpp` | 167 | **Power sequencing**: giữ chân EN 1s (BT vẫn chạy). `Power_Update()` đọc ADC mỗi 1s. Auto shutdown 60s |
| 8 | `HAL_Buzz.cpp` | 60 | **LEDC + RTOS task**: BuzzerThread (prio 1) loop đọc duration + freq → `ledcWriteTone` |
| 9 | `HAL_Audio.cpp` | 37 | **TonePlayer**: map tên bài → MusicCode array. Callback gọi `Buzz_Tone()` |
| 10 | `HAL_SdCard.cpp` | 67 | **SPI**: HSPI riêng (80MHz), SD.begin(15,...). `SD_GetReady()` = digitalRead(DET_PIN) |
| 11 | `HAL_I2C_Scan.cpp` | 45 | **I2C scan**: quét address 1–127, in kết quả ra Serial |
| 12 | `HAL_Bluetooth.cpp` | 284 | **NimBLE**: scan "Dummy-Robot" → connect → subscribe notification. Callbacks: connect, disconnect, auth |
| 13 | `lv_port_disp.cpp` | 74 | **SPI LCD flush**: `TFT_eSPI::pushColors()` — chính là nơi LVGL đẩy pixel ra màn hình |
| 14 | `lv_port_fatfs.cpp` | 350 | **FatFS + LVGL**: driver "S:" cho SD card. `f_open/f_read/f_write` wrapper |
| 15 | `ButtonEvent.h` | 131 | **Button driver**: state machine (PRESS→LONG_PRESS→RELEASE). Events: pressed, released, short click, long press, double click |
| 16 | `TonePlayer.h` | — | **TonePlayer**: play melody non-blocking (update tick). Set callback → Buzz_Tone |
| 17 | `StorageService.h` | 67 | **JSON config**: Add(key, value, type) → SaveFile/LoadFile (ArduinoJson). Types: INT, FLOAT, DOUBLE, STRING |
| 18 | `HysteresisFilter.h` | — | **Hysteresis filter**: chỉ thay đổi output khi input vượt ngưỡng | |
| 19 | `MedianQueueFilter.h` | — | **Median queue filter**: lấy median của N mẫu cuối |

### Sơ đồ Hardware

```
ESP32-PICO-D4
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  I2C (SDA=32, SCL=33)                                       │
│  └── MPU6050/9250 (0x68) ─── IMU 9-axis                    │
│                                                             │
│  SPI2 (HSPI)                                                 │
│  ├── SD Card (CS=15, MOSI=PB15, MISO=PB14, SCK=PB13)      │
│  │   └── FatFS + LVGL FS driver "S:"                       │
│  └── ST7789 (CS=PB0, DC=PA4, RST=PA6, SCK=PA5, MOSI=PA7) │
│      └── TFT_eSPI + LVGL display driver                    │
│                                                             │
│  LEDC (PWM)                                                  │
│  ├── Backlight (BLK=12, ch=0, 5kHz, 10-bit)                │
│  └── Buzzer (BUZZ=25, ch=2)                                 │
│                                                             │
│  GPIO                                                        │
│  ├── Encoder A=35 (ISR), B=34, Push=27                     │
│  ├── Power EN=21, BAT_DET=37, CHG_DET=38                   │
│  └── SD DET=22                                              │
│                                                             │
│  UART: Serial @ 115200                                      │
│  BLE: NimBLE                                                │
└─────────────────────────────────────────────────────────────┘
```

### Mục tiêu cần đạt

Sau Phase 5, bạn trả lời được:

- [x] INIT thứ tự driver thế nào? → BT → Power (1s) → Backlight → Encoder → Buzz → Audio → SD → I2C → IMU
- [x] Mỗi driver dùng hardware gì?
  - Backlight: LEDC PWM 5kHz 10-bit
  - IMU: I2C MPU9250 @ 0x68
  - Power: ADC (read voltage) + GPIO (EN, CHG_DET)
  - Encoder: GPIO interrupt CHANGE
  - Buzz: LEDC PWM + RTOS thread
  - Audio: TonePlayer → Buzz callback
  - SD: SPI (HSPI) 80MHz
  - BT: NimBLE BLE
- [x] Pin mapping
  - LCD: CS=PB0, DC=PA4, RST=PA6, SCK=PA5, MOSI=PA7, BLK=12
  - Encoder: A=35, B=34, Push=27
  - I2C: SDA=32, SCL=33
  - SD: CS=15, MOSI=PB15, MISO=PB14, SCK=PB13, DET=22
- [x] API của mỗi driver
- [x] Driver giao tiếp với App thế nào? → AccountSystem Commit/Pull/Notify
- [x] Filter dùng ở đâu? → ACT_Power (Hysteresis + MedianQueue cho battery)
- [x] BLE kết nối với ai? → Dummy-Robot (service UUID 0001, characteristic 1001)
- [x] LCD refresh? → TFT_eSPI SPI 80MHz, pushColors half-screen 240×120
- [x] SD tốc độ? → HSPI 80MHz

---

## Bonus: File cần đọc cho từng vai trò

### Nếu bạn muốn thêm Page mới

| File | Để hiểu |
|---|---|
| `Pages/_Template/Template.h/.cpp` | Boilerplate page mẫu |
| `Pages/_Template/TemplateView.cpp` | Cách tạo LVGL widgets |
| `Pages/_Template/TemplateModel.cpp` | Cách kết nối Account |
| `Pages/AppFactory.cpp` | Đăng ký page vào factory |
| `App/App.cpp` | Install page |
| `Utils/PageManager/PageBase.h` | Base class |

### Nếu bạn muốn thêm Account node mới

| File | Để hiểu |
|---|---|
| `Accounts/_ACT_LIST.inc` | Đăng ký node |
| `Accounts/ACT_Def.h` | Định nghĩa message struct |
| `Accounts/ACT_IMU.cpp` | Publisher node (commit/publish) |
| `Accounts/ACT_Power.cpp` | Pull + Timer node |
| `Accounts/Account_Master.h/.cpp` | Init của account system |
| `Utils/AccountSystem/Account.h` | Class Account |

### Nếu bạn muốn thêm driver mới

| File | Để hiểu |
|---|---|
| `HAL/HAL.h` | Thêm API vào namespace |
| `HAL/HAL.cpp` | Thêm Init() + Update() |
| `HAL/HAL_IMU.cpp` | Driver mẫu (I2C + Commit) |
| `HAL/HAL_Encoder.cpp` | Driver mẫu (GPIO ISR + Account) |
| `App/Configs/Config.h` | Thêm pin definitions |

### Nếu bạn muốn sửa LVGL UI

| File | Để hiểu |
|---|---|
| `Pages/StartUp/StartUpView.cpp` | Animation timeline |
| `Pages/SystemInfos/SystemInfosView.cpp` | Style + Group + Flex layout |
| `Pages/StatusBar/StatusBar.cpp` | Timer update + LVGL styles |
| `Resources/ResourcePool.cpp` | Import fonts/images |
| `Port/lv_port/lv_port_disp.cpp` | Display buffer |

---

## Tổng kết roadmap

```
Phase 1: Kiến trúc (30 phút)
  ├── Đọc: 9 files
  ├── Hiểu: 3 lớp, boot sequence, pin map
  └── Output: biết file nào làm gì

Phase 2: GUI (1-2 giờ)
  ├── Đọc: 16 files
  ├── Hiểu: MVC, PageManager, LVGL port, animation
  └── Output: biết cách thêm page mới

Phase 3: RTOS (45-60 phút)
  ├── Đọc: 9 files
  ├── Hiểu: 5 tasks, priority, ISR, task notification
  └── Output: biết task nào chạy gì

Phase 4: Message (1.5-2 giờ)
  ├── Đọc: 16 files
  ├── Hiểu: AccountSystem, Pub/Sub, PingPongBuffer
  └── Output: biết data flow giữa các module

Phase 5: Driver (2-3 giờ)
  ├── Đọc: 19 files
  ├── Hiểu: 9 drivers, interrupt, PWM, I2C, SPI, ADC, BLE
  └── Output: đọc được và sửa được bất kỳ driver nào

Tổng: 69 files, ~6-9 giờ
```
