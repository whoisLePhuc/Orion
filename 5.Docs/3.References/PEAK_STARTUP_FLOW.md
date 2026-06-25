# 🔷 Peak Project — Startup Flow Analysis

> **Mục tiêu**: Trace chi tiết toàn bộ luồng khởi tạo từ lúc ESP32 boot cho đến khi GUI hiển thị.
>
> **Người đọc**: Developer muốn hiểu nên đọc file nào trước khi bắt đầu contribute.

---

## 1. Chương trình bắt đầu từ đâu?

**Entry point**: `src/main.cpp`

```
Peak-ESP32-fw/src/
├── main.cpp                 ← điểm vào
├── HAL/                     ← hardware abstraction
├── Port/                    ← LVGL port + display
└── App/                     ← application layer
    ├── App.h/cpp
    ├── Configs/
    ├── Accounts/
    ├── Pages/
    ├── Resources/
    └── Utils/
```

Với Arduino framework trên ESP32, `main()` được ẩn bên trong core, nó sẽ gọi:
1. `init()` — khởi tạo ESP32 hardware (clock, GPIO, FreeRTOS)
2. `initVariant()` — variant-specific init
3. `setup()` — **code của người dùng** (Peak's entry)
4. `loop()` — **vòng lặp vô hạn** (Peak's main loop)

> **Arduino entry point** được định nghĩa trong `cores/esp32/main.cpp` của ESP32 Arduino core. Nó tạo một FreeRTOS task `loopTask` chạy `setup()` + `loop()`. Không cần quan tâm lớp này — chỉ cần biết `setup()` là nơi bắt đầu.

---

## 2. Hàm `main.cpp` — `setup()` và `loop()`

```cpp
// src/main.cpp
void setup()
{
    HAL::Init();        // (1) Khởi tạo phần cứng
    Port_Init();        // (2) Khởi tạo màn hình + LVGL
    App_Init();         // (3) Khởi tạo ứng dụng
}

void loop()
{
    HAL::Update();      // Cập nhật các cảm biến định kỳ
    delay(20);          // ~50Hz
}
```

> **Lưu ý quan trọng**: LVGL rendering KHÔNG chạy trong `loop()`. Nó chạy trong một **FreeRTOS task riêng** được tạo trong `Port_Init()`.

---

## 3. Thứ tự khởi tạo các module (chi tiết từng bước)

### Phase 1: `HAL::Init()` — Khởi tạo Hardware Abstraction Layer

```
HAL::Init()
├── Serial.begin(115200)
├── In ra thông tin firmware: [littleVisual] v1.0, Author PZH
│
├── lv_disp_buf_p = malloc(DISP_BUF_SIZE * sizeof(lv_color_t))
│   └── DISP_BUF_SIZE = 240 × 120 = 28800 pixel (half screen)
│   └── Cấp phát SỚM để chiếm vùng heap lớn nhất trước
│
├── HAL::BT_Init()
│   └── NimBLEDevice::init("Peak-BLE")
│   └── Tạo BLE scan để tìm Dummy-Robot
│   └── ⚠️ Gọi TRƯỚC vì các init sau có thể gây nhiễu BLE
│
├── HAL::Power_Init()
│   ├── PinMode(CONFIG_BAT_CHG_DET_PIN)
│   ├── PinMode(CONFIG_POWER_EN_PIN, OUTPUT), digitalWrite LOW
│   ├── Chờ 1 giây (while loop bên trong gọi BT_Update)
│   ├── digitalWrite(CONFIG_POWER_EN_PIN, HIGH)  ← BẬT nguồn chính
│   ├── Power_ADC_Init() — pin ADC đọc pin
│   └── SetAutoLowPowerTimeout(60s), disable
│
├── HAL::Backlight_Init()
│   └── ledcSetup(0, 5000, 10) — PWM 5kHz, 10-bit
│   └── ledcAttachPin(CONFIG_SCREEN_BLK_PIN, 0)
│   └── ledcWrite(0, 0) — MÀN HÌNH TẮT (vẫn chưa có gì để hiển thị)
│
├── HAL::Encoder_Init()
│   ├── PinMode encoder A, B (INPUT_PULLUP), Push (INPUT_PULLUP)
│   ├── attachInterrupt(Encoder_A, Encoder_IrqHandler, CHANGE)
│   ├── ButtonEvent khởi tạo với 5000ms long press
│   └── Tạo Account "Encoder" (⚠️ tạo account trước Accounts_Init!)
│    → Xem PM_Router.cpp và PM_Base.cpp để hiểu PageManager lifecycle
│
├── HAL::Buzz_init()
│   └── ledcSetup(2, 5000, 10), ledcAttachPin(CONFIG_BUZZ_PIN, 2)
│
├── HAL::Audio_Init()
│   └── Khởi tạo I2S / DAC để play WAV từ SD
│
├── HAL::SD_Init()
│   ├── SPI bus riêng (HSPI) cho SD card
│   ├── SD.begin(15, *sd_spi, 80000000) — 80MHz SPI
│   └── In card type + size
│
├── HAL::I2C_Init(true)
│   ├── Wire.begin(CONFIG_MCU_SDA_PIN, CONFIG_MCU_SCL_PIN)
│   └── Nếu startScan = true: scan I2C bus, in địa chỉ device
│
├── HAL::IMU_Init()
│   └── MPU9250.setup(0x68) — kiểm tra kết nối I2C
│
└── HAL::Audio_PlayMusic("Startup")
    └── Gửi notify tới MusicPlayer account → play startup sound
```

> **Các file cần đọc**: `HAL/HAL.cpp`, `HAL/HAL_Power.cpp`, `HAL/HAL_Backlight.cpp`, `HAL/HAL_Encoder.cpp`, `HAL/HAL_IMU.cpp`, `HAL/HAL_SdCard.cpp`, `HAL/HAL_Bluetooth.cpp`

### Phase 2: `Port_Init()` — Khởi tạo Display + LVGL

```
Port_Init()                           ← Display.cpp
├── static SCREEN_CLASS screen (TFT_eSPI)
├── screen.begin()                    ← Khởi tạo SPI LCD
├── screen.setRotation(0)
├── screen.fillScreen(TFT_BLACK)      ← Màn hình đen
│
├── lv_init()                         ← Khởi tạo LVGL (cấu hình, allocator)
│
├── lv_port_disp_init(&screen)        ← lv_port_disp.cpp
│   ├── lv_log_register_print_cb(my_print)  ← log LVGL ra Serial
│   ├── lv_disp_draw_buf_init(&disp_buf, lv_disp_buf_p, NULL, DISP_BUF_SIZE)
│   │   └── Dùng buffer đã malloc trong HAL::Init()
│   ├── lv_disp_drv_init → hor=240, ver=240, flush_cb=disp_flush_cb
│   └── lv_disp_drv_register          ← Đăng ký display driver
│
├── lv_port_indev_init()              ← lv_port_indev.cpp
│   └── lv_indev_drv_init → type=ENCODER, read_cb=encoder_read
│   └── encoder_read() gọi HAL::Encoder_GetDiff() + Encoder_GetIsPush()
│
├── lv_fs_if_init()                   ← lv_port_fatfs.cpp
│   └── Đăng ký driver "S:" cho file system (SD card)
│
├── xTaskCreate(TaskLvglUpdate, "LvglThread", 20000, ...,
│               configMAX_PRIORITIES - 1, &handleTaskLvgl)
│   └── Tạo FreeRTOS task cho LVGL rendering
│   └── Task sẽ chờ ulTaskNotifyTake() — bị BLOCK chờ signal
│   └── Sau signal: lv_task_handler() + delay(5) trong vòng lặp
│
└── HAL::Backlight_SetGradual(500, 1000)  ← BẬT đèn nền từ từ (0→500, 1s)
```

> **Các file cần đọc**: `Port/Display.cpp`, `Port/lv_port/lv_port_disp.cpp`, `Port/lv_port/lv_port_indev.cpp`, `Port/lv_port/lv_port_fatfs.cpp`

### Phase 3: `App_Init()` — Khởi tạo Application Layer

```
App_Init()                            ← App/App.cpp
│
├── static AppFactory factory
│   └── AppFactory kế thừa PageFactory
│   └── CreatePage(name) tạo: Template, SystemInfos, Startup
│
├── static PageManager manager(&factory)
│   └── Constructor: SetGlobalLoadAnimType(mặc định OVER_LEFT)
│
├── Accounts_Init()                   ← App/Accounts/Account_Master.cpp
│   ├── static AccountBroker dataCenter("MASTER")
│   │
│   ├── [Pass 1] Tạo các Account nodes từ _ACT_LIST.inc:
│   │   ├── Account("Storage",     MASTER, 0)                ← ACT_Storage
│   │   ├── Account("Power",       MASTER, 0)                ← ACT_Power
│   │   ├── Account("IMU",         MASTER, sizeof(IMU_Info_t)) ← ACT_IMU
│   │   ├── Account("StatusBar",   MASTER, 0)                ← ACT_StatusBar
│   │   ├── Account("MusicPlayer", MASTER, 0)                ← ACT_MusicPlayer
│   │   └── Account("SysConfig",   MASTER, 0)                ← ACT_SysConfig
│   │
│   └── [Pass 2] Gọi _ACT_*_Init() cho từng account:
│       ├── ACT_Storage.cpp:   SetEventCallback, Subscribe("SysConfig")
│       ├── ACT_Power.cpp:     SetEventCallback, Subscribe("MusicPlayer"), SetTimerPeriod(500)
│       ├── ACT_IMU.cpp:       chỉ gán account global (IMU_Commit dùng sau)
│       ├── ACT_StatusBar.cpp: SetEventCallback, Subscribe("Power"), Subscribe("Storage")
│       ├── ACT_MusicPlayer.cpp: SetEventCallback
│       └── ACT_SysConfig.cpp: SetEventCallback, Subscribe("Storage")
│                               + STORAGE_VALUE_REG(6 biến cấu hình)
│
├── Resource.Init()                   ← App/Resources/ResourcePool.cpp
│   ├── lv_obj_remove_style_all(lv_scr_act())  ← Xóa style mặc định
│   ├── lv_disp_set_bg_color(default, black)
│   ├── Font_.SetDefault(&lv_font_montserrat_14)
│   └── Resource_Init():
│       ├── Import fonts (5): bahnschrift_13/17/32/65, agencyb_36
│       └── Import images (27): alarm, battery, compass, gps_arrow*, ...
│
├── StatusBar::Init(lv_layer_top())   ← App/Pages/StatusBar/StatusBar.cpp
│   └── StatusBar_Create(lv_layer_top())
│       ├── Tạo container 240×22, y=-22 (trên màn hình, chờ appear)
│       ├── Tạo icon satellite, SD, Bluetooth, battery, clock, recorder label
│       ├── SetStyle(TRANSP)
│       └── lv_timer_create(StatusBar_Update, 1000ms)
│           └── Timer pull "Storage" + "Power" để cập nhật UI
│
├── manager.Install("Template", "Pages/Template")
│   └── Factory.CreatePage("Template") → new Page::Template
│   └── base->onCustomAttrConfig()    ← SetCustomCache ON, LOAD_ANIM_OVER_BOTTOM
│   └── Register vào PagePool
│
├── manager.Install("SystemInfos", "Pages/SystemInfos")
│   └── Factory.CreatePage("SystemInfos") → new Page::SystemInfos
│   └── Register vào PagePool
│
├── manager.Install("Startup", "Pages/Startup")
│   └── Factory.CreatePage("Startup") → new Page::Startup
│   └── onCustomAttrConfig()          ← Disable cache, LOAD_ANIM_NONE
│   └── Register vào PagePool
│
├── manager.SetGlobalLoadAnimType(OVER_TOP, 500ms)
│   └── Global animation: page mới đè từ trên xuống
│
├── manager.Push("Pages/Startup")     ← CHÌA KHÓA: Khởi chạy page đầu tiên!
│   └── (Xem chi tiết Section 4)
│
├── ACCOUNT_SEND_NOTIFY_CMD(Storage, STORAGE_CMD_LOAD)
│   └── AccountMaster.Notify("Storage") → Storage onEvent(LOAD)
│   └── storageService.LoadFile()
│   └── Pull("SysConfig") → MapConv set dir + level range
│
├── ACCOUNT_SEND_NOTIFY_CMD(SysConfig, SYSCONFIG_CMD_LOAD)
│   └── AccountMaster.Notify("SysConfig") → SysConfig onEvent(LOAD)
│   └── HAL::Buzz_SetEnable(sysConfig.soundEnable)
│
└── INIT_DONE()
    └── [ARDUINO] xTaskNotifyGive(handleTaskLvgl)
        └── WAKE UP LVGL task! (lúc này LVGL mới bắt đầu render)
```

> **Các file cần đọc**: `App/App.cpp`, `App/Accounts/Account_Master.cpp`, `App/Accounts/_ACT_LIST.inc`, `App/Resources/ResourcePool.cpp`, `App/Pages/AppFactory.cpp`, `App/Pages/StatusBar/StatusBar.cpp`

---

## 4. Luồng thực thi từ boot đến khi GUI xuất hiện

### Timeline chi tiết (theo thứ tự thời gian)

```
ESP32 Power On / Reset
│
├── [0ms]   ESP32 BootROM → Flash bootloader → Arduino core init
│           - CPU clock: 240MHz
│           - FreeRTOS kernel khởi động
│           - Tạo task `loopTask` (Arduino core)
│
├── [~10ms] void setup() được gọi
│
├── [10ms]  HAL::Init()
│   ├── Serial (115200) — in firmware info
│   ├── malloc display buffer (115200 bytes)  ← chiếm heap lớn
│   ├── BT_Init() — BLE scan start
│   ├── Power_Init() — chờ 1s giữ nguồn, rồi bật
│   │   └── BLE vẫn scan trong khi chờ
│   ├── Backlight_Init() — PWM setup, màn hình tối
│   ├── Encoder_Init() — GPIO interrupt
│   ├── Buzz_init() — PWM
│   ├── Audio_Init()
│   ├── SD_Init() — SPI SD card detect
│   ├── I2C_Init(true) — scan bus I2C
│   ├── IMU_Init() — check MPU9250
│   └── Audio_PlayMusic("Startup") — phát nhạc khởi động
│
├── [~1100ms] Port_Init()
│   ├── screen.begin() — TFT LCD SPI init
│   ├── lv_init() — LVGL internal init
│   ├── lv_port_disp_init() — đăng ký display + buffer
│   ├── lv_port_indev_init() — đăng ký encoder input
│   ├── lv_fs_if_init() — đăng ký SD card file system
│   ├── xTaskCreate(TaskLvglUpdate)
│   │   └── Task LVGL được tạo NHƯNG blocked (chờ notify)
│   └── Backlight_SetGradual(500, 1000) — màn hình sáng dần
│
├── [~1150ms] App_Init()
│   ├── Accounts_Init()
│   │   └── 6 Account nodes được tạo + đăng ký callback
│   ├── Resource.Init()
│   │   └── 5 fonts + 27 images import vào ResourcePool
│   ├── StatusBar::Init()
│   │   └── UI status bar (ẩn phía trên, y=-22)
│   ├── manager.Install × 3 (Template, SystemInfos, Startup)
│   │   └── Gọi onCustomAttrConfig() cho từng page
│   ├── manager.SetGlobalLoadAnimType(OVER_TOP, 500)
│   ├── manager.Push("Pages/Startup") ★
│   │   ├── SwitchTo() → PageCurrent = Startup
│   │   ├── PageCurrent->priv.State = PAGE_STATE_LOAD
│   │   ├── StateUpdate(PageCurrent):
│   │   │   [LOAD]     lv_obj_create(root)  ← root LVGL object
│   │   │             base->onViewLoad() gọi:
│   │   │               ├── Model.Init() → tạo Account("StartupModel")
│   │   │               ├── Model.SetEncoderEnable(false)
│   │   │               ├── View.Create(root)
│   │   │               │   ├── Container + label "DUMMY"
│   │   │               │   └── Animation timeline (width 0→110, y slide)
│   │   │               └── lv_timer_create(onTimer, 2000, repeat=1)
│   │   │             base->onViewDidLoad() → fade out root (300ms delay, 1500ms)
│   │   │   [WILL_APPEAR] base->onViewWillAppear() → start animation
│   │   │                 lv_obj_clear_flag(root, HIDDEN) ← UI visible!
│   │   │                 SwitchAnimCreate() → animation OVER_TOP
│   │   │   [DID_APPEAR] → onViewDidAppear()
│   │   └── [ACTIVITY] Page ổn định, timer bắt đầu đếm 2 giây
│   │
│   ├── Notify Storage LOAD (tải config từ SD)
│   ├── Notify SysConfig LOAD (áp dụng cấu hình)
│   │
│   └── INIT_DONE() → xTaskNotifyGive(handleTaskLvgl)
│       └── Task LVGL BẮT ĐẦU chạy: lv_task_handler() loop
│           └── LVGL xử lý animation, timer, rendering
│
│   ═══════════════════════════════════════════════════
│   🎬 GUI XUẤT HIỆN — MÀN HÌNH STARTUP
│   ═══════════════════════════════════════════════════
│
├── [~3300ms] Timer Startup.onTimer() fire (sau 2s)
│   └── manager.Push("Pages/Template")
│       ├── StateUpdate(Startup):
│       │   [ACTIVITY → WILL_DISAPPEAR]
│       │     onViewWillDisappear()
│       │     SwitchAnimCreate() → animation exit
│       │   [DID_DISAPPEAR]
│       │     onViewDidDisappear()
│       │       └── StatusBar::Appear(true) ← thanh trạng thái trượt xuống
│       │     Startup cached? No → [UNLOAD] → onViewDidUnload()
│       │       └── Model.DeInit(), SetEncoderEnable(true)
│       │     root bị xóa (lv_obj_del_async)
│       │
│       └── StateUpdate(Template):
│           [LOAD]     onViewLoad() → View.Create(root)
│           [WILL_APPEAR] onViewWillAppear() → timer 100ms
│           [DID_APPEAR → ACTIVITY]
│
├── [~4000ms+] void loop() chạy mãi mãi
│    ├── HAL::Update()
│    │   ├── Power_Update()   — ADC read battery (1s interval)
│    │   ├── Encoder_Update() — button event monitor
│    │   ├── Audio_Update()   — music player update
│    │   ├── IMU_Update()     — I2C read MPU → Commit → AccountSystem
│    │   ├── BT_Update()      — BLE connect if device found
│    │   └── SD_Update()      — (500ms interval)
│    │
│    └── delay(20) — ~50Hz loop
│
└── LVGL Task chạy song song:
    └── for(;;) { lv_task_handler(); delay(5); }
        ├── Xử lý LVGL timers (StatusBar update 1s, animation)
        ├── Xử lý input (encoder diff/push)
        ├── Redraw các vùng dirty
        └── disp_flush_cb() → TFT_eSPI pushColors() SPI
```

---

## 5. Startup flow dạng text

```
ESP32 BOOT
    │
    ▼
setup()
    │
    ├── HAL::Init() ──────────────────────────────────────────────┐
    │   ├─ Serial                in logo "littleVisual v1.0"     │
    │   ├─ malloc                cấp display buffer 115KB        │
    │   ├─ BT_Init               BLE scan (tìm Dummy-Robot)      │
    │   ├─ Power_Init            giữ nguồn 1s, bật nguồn        │
    │   ├─ Backlight_Init        PWM 0% (màn hình tối)          │
    │   ├─ Encoder_Init          GPIO interrupt                  │
    │   ├─ Buzz_init             PWM buzzer                      │
    │   ├─ Audio_Init            I2S/DAC                         │
    │   ├─ SD_Init               HSPI, 80MHz                    │
    │   ├─ I2C_Init(true)        scan I2C bus                   │
    │   ├─ IMU_Init              MPU9250@0x68                   │
    │   └─ Audio_PlayMusic      "Startup" sound                 │
    │                                                           │
    ├── Port_Init() ────────────────────────────────────────────┤
    │   ├─ screen.begin()        TFT SPI init                   │
    │   ├─ lv_init()             LVGL core init                 │
    │   ├─ lv_port_disp_init     display driver + buffer         │
    │   ├─ lv_port_indev_init    encoder input device            │
    │   ├─ lv_fs_if_init         "S:" SD file system             │
    │   ├─ xTaskCreate           LvglThread (BLOCKED)            │
    │   └─ Backlight_SetGradual  sáng dần 0→500 (1s)            │
    │                                                           │
    └── App_Init() ─────────────────────────────────────────────┤
        ├─ Accounts_Init() ─────────────────────────────────┐    │
        │   Storage, Power, IMU, StatusBar,                 │    │
        │   MusicPlayer, SysConfig                          │    │
        │   ← 6 Account nodes created, callbacks set        │    │
        └───────────────────────────────────────────────────┘    │
        ├─ Resource.Init()                                      │
        │   5 fonts + 27 images imported                         │
        ├─ StatusBar::Init()                                    │
        │   UI StatusBar created (y=-22, chờ appear)            │
        ├─ manager.Install("Template")                         │
        │   → onCustomAttrConfig()                              │
        ├─ manager.Install("SystemInfos")                      │
        │   → onCustomAttrConfig()                              │
        ├─ manager.Install("Startup")                          │
        │   → onCustomAttrConfig()                              │
        │                                                       │
        ├── manager.Push("Pages/Startup") ★                    │
        │   │                                                   │
        │   └── PageManager::SwitchTo()                        │
        │       │                                               │
        │       ├── State: LOAD ─────────────────────────┐     │
        │       │   onViewLoad()                          │     │
        │       │   ├─ Model.Init()                       │     │
        │       │   ├─ View.Create(root)                  │     │
        │       │   │   "DUMMY" label + animation         │     │
        │       │   └─ timer 2000ms (auto next page)      │     │
        │       │   onViewDidLoad() → fade out root       │     │
        │       └─────────────────────────────────────────┘     │
        │       │                                               │
        │       ├── State: WILL_APPEAR ────────────────┐       │
        │       │   onViewWillAppear()                 │       │
        │       │   → start animation timeline          │       │
        │       │   → root visible!                    │       │
        │       │   → OVER_TOP animation (500ms)       │       │
        │       └──────────────────────────────────────┘       │
        │       │                                               │
        │       ├── 🎬 STARTUP UI HIỂN THỊ                     │
        │       │                                               │
        │       ├── State: DID_APPEAR → ACTIVITY               │
        │       │   onViewDidAppear()                           │
        │       │   → timer 2s đếm ngược                        │
        │       │                                               │
        │       ├── After 2s: onTimer fire ★                    │
        │       │   manager.Push("Pages/Template")              │
        │       │   │                                           │
        │       │   └── Startup cleanup:                        │
        │       │       ├── WILL_DISAPPEAR → onViewWillDisappear│
        │       │       ├── SwitchAnimCreate (exit)             │
        │       │       ├── DID_DISAPPEAR → onViewDidDisappear  │
        │       │       │   └── StatusBar::Appear(true)         │
        │       │       └── UNLOAD → onViewDidUnload           │
        │       │           Model.DeInit(), Encoder enable      │
        │       │           root deleted                         │
        │       │                                               │
        │       └── Template init:                              │
        │           ├── LOAD → onViewLoad()                     │
        │           ├── WILL_APPEAR → animation                 │
        │           ├── DID_APPEAR → timer 100ms update         │
        │           └── 🎬 TEMPLATE PAGE HIỂN THỊ               │
        │                                                       │
        ├── Notify Storage LOAD (config from SD)               │
        ├── Notify SysConfig LOAD (apply sound config)         │
        │                                                       │
        └── INIT_DONE() ───────────────────────────────┐        │
            xTaskNotifyGive(handleTaskLvgl)              │        │
            → LVGL Task bắt đầu chạy!                   │        │
            └───────────────────────────────────────────┘        │
                                                                  │
    loop() ───────────────────────────────────────────────────────┘
    ├── HAL::Update()
    │   Power → Encoder → Audio → IMU → BT → SD
    └── delay(20)
```

---

## 6. Các FreeRTOS task được tạo

| Task Name | Stack | Priority | File | Vai trò |
|---|---|---|---|---|
| **loopTask** (Arduino) | mặc định | 1 | Arduino core | Chạy `setup()` + `loop()` của Peak |
| **LvglThread** | 20000 words (~80KB) | `configMAX_PRIORITIES-1` (cao nhất) | `Port/Display.cpp` | LVGL rendering: `lv_task_handler()` + `delay(5)` |
| **Idle Task** | — | 0 | FreeRTOS | Task rỗi của FreeRTOS |
| **BLE stack tasks** | — | — | NimBLE | Task nội bộ của NimBLE stack |

> **Lưu ý**: `loopTask` (Arduino) và `LvglThread` **chạy song song** trên 2 nhân của ESP32. LVGL task có priority cao nhất để đảm bảo rendering mượt. Arduino loop chạy ở priority thấp hơn.

---

## 7. File map — Tài liệu nên đọc theo thứ tự

Muốn hiểu startup flow, đọc các file **theo thứ tự sau**:

| Thứ tự | File | Lý do |
|---|---|---|
| 1 | `src/main.cpp` | Entry point — 3 dòng gọi hàm chính |
| 2 | `src/HAL/HAL.cpp` | Khởi tạo toàn bộ phần cứng |
| 3 | `src/HAL/HAL.h` | API HAL layer — tất cả các hàm Init/Update |
| 4 | `src/HAL/HAL_Power.cpp` | Power sequencing quan trọng (giữ nguồn 1s) |
| 5 | `src/HAL/HAL_Encoder.cpp` | Encoder + ButtonEvent + Account "Encoder" |
| 6 | `src/HAL/HAL_IMU.cpp` | IMU init + Commit data vào Account system |
| 7 | `src/HAL/HAL_Bluetooth.cpp` | BLE init (được gọi sớm nhất trong HAL) |
| 8 | `src/Port/Display.cpp` | Khởi tạo màn hình, LVGL, và tạo LvglThread |
| 9 | `src/Port/lv_port/lv_port_disp.cpp` | LVGL display driver + buffer setup |
| 10 | `src/Port/lv_port/lv_port_indev.cpp` | LVGL input driver (encoder) |
| 11 | `src/App/App.cpp` | Application init — Account, Resource, Pages |
| 12 | `src/App/App.h` | Macro INIT_DONE(), ACCOUNT_SEND_NOTIFY_CMD |
| 13 | `src/App/Accounts/Account_Master.cpp` | Account system khởi tạo |
| 14 | `src/App/Accounts/_ACT_LIST.inc` | Danh sách 6 Account nodes |
| 15 | `src/App/Accounts/ACT_Def.h` | Định nghĩa cấu trúc dữ liệu cho từng Account |
| 16 | `src/App/Utils/AccountSystem/Account.h` | Core Account class (Pub/Sub) |
| 17 | `src/App/Utils/AccountSystem/AccountBroker.h` | Account registry center |
| 18 | `src/App/Resources/ResourcePool.cpp` | Font + Image imports |
| 19 | `src/App/Pages/AppFactory.cpp` | Page factory — tạo instance page từ tên |
| 20 | `src/App/Pages/StatusBar/StatusBar.cpp` | Status bar UI (top layer) |
| 21 | `src/App/Utils/PageManager/PageManager.h` | PageManager API |
| 22 | `src/App/Utils/PageManager/PM_Router.cpp` | `Push()`, `Pop()`, `SwitchTo()` logic |
| 23 | `src/App/Utils/PageManager/PM_State.cpp` | State machine (LOAD→WILL_APPEAR→...) |
| 24 | `src/App/Utils/PageManager/PM_Base.cpp` | `Install()`, `Register()`, page pool |
| 25 | `src/App/Utils/PageManager/PageBase.h` | Base class cho mọi page |
| 26 | `src/App/Pages/StartUp/StartUp.cpp` | Page đầu tiên — startup screen |
| 27 | `src/App/Pages/StartUp/StartUpView.cpp` | UI của startup screen |
| 28 | `src/App/Pages/StartUp/StartUpModel.cpp` | Model của startup (Account) |
| 29 | `src/App/Pages/_Template/Template.cpp` | Page thứ hai (sau startup timer) |
| 30 | `src/HAL/CommonMacro.h` | Các macro utility (IntervalExecute, v.v.) |

---

## 8. Tóm tắt

```
setup() [main.cpp]
  │
  ├── HAL::Init()          ← Khởi tạo tất cả drivers (IMU, SD, BT, Power...)
  │   └── Quan trọng: malloc display buffer SỚM để chiếm heap liên tục
  │   └── BT init sớm nhất để tránh nhiễu từ các init khác
  │
  ├── Port_Init()          ← Khởi tạo màn hình + LVGL
  │   ├── screen.begin()   ← TFT SPI
  │   ├── lv_init()        ← LVGL core
  │   └── xTaskCreate(LvglThread) ← Task LVGL chờ signal
  │
  └── App_Init()           ← Khởi tạo ứng dụng
      ├── Accounts_Init()  ← 6 Account nodes (Pub/Sub system)
      ├── Resource.Init()  ← Font + Image
      ├── StatusBar::Init() ← UI overlay
      ├── manager.Push("Pages/Startup") ← Page lifecycle bắt đầu
      ├── Notify Storage/SysConfig LOAD
      └── INIT_DONE() → LVGL task wake → LVGL bắt đầu render
```

**Điểm chốt**: `INIT_DONE()` (gọi `xTaskNotifyGive`) là **mốc quan trọng nhất** — trước đó LVGL task bị block, sau đó LVGL mới thực sự render animation và UI. Màn hình startup xuất hiện khoảng **1.1–1.5 giây** sau khi boot.
