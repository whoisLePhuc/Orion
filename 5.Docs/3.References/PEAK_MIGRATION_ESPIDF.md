# 🔷 Peak — Migration Guide: Arduino → ESP-IDF (ESP32-S3)

> Tài liệu chi tiết chuyển đổi Peak từ nền tảng **ESP32-PICO-D4 + Arduino Framework**
> sang **ESP32-S3 + ESP-IDF v5.x**.
>
> Mục tiêu: tận dụng tối đa code hiện có, chỉ viết lại phần phụ thuộc Arduino.

---

## 1. So sánh nền tảng

| Tiêu chí | Hiện tại (Peak) | Mục tiêu |
|---|---|---|
| **MCU** | ESP32-PICO-D4 (LX6) | ESP32-S3 (LX7) |
| **Core** | 2 × Xtensa LX6 @ 240MHz | 2 × Xtensa LX7 @ 240MHz |
| **SRAM** | 520KB | 512KB (+ up to 2MB PSRAM) |
| **Flash** | 4MB | 8MB+ |
| **PSRAM** | ❌ Không | ✅ Hỗ trợ Octal SPI PSRAM |
| **Framework** | Arduino (ESP32 Core) | ESP-IDF v5.x |
| **Build** | PlatformIO | CMake + idf.py |
| **FreeRTOS** | Arduino wrapper | Native ESP-IDF FreeRTOS SMP |
| **LCD** | ST7789 240×240, SPI | ST7789 240×240, SPI |
| **IMU** | MPU6050/9250 (I2C) | MPU6050/9250 (I2C) |
| **Tính khả dụng** | Arduino libraries | ESP-IDF components |

### Lợi thế ESP32-S3 so với ESP32-PICO-D4

```
ESP32-PICO-D4              ESP32-S3
┌──────────────┐          ┌──────────────────┐
│ SRAM: 520KB  │          │ SRAM: 512KB       │
│ Flash: 4MB   │          │ Flash: 8MB+        │
│ PSRAM: ❌    │          │ PSRAM: ✅ 2MB     │ ← LVGL buffer lớn hơn!
│               │          │                   │
│ LCD buffer:   │          │ LCD buffer:       │
│ 240×120×2     │          │ 240×240×2 (full)  │ ← Full screen double buffer!
│ = 57.6KB     │          │ = 115.2KB         │
└──────────────┘          └──────────────────┘
```

---

## 2. Những phần có thể giữ nguyên

### ✅ Framework Core — Giữ 100%

| Module | Files | Lý do |
|---|---|---|
| **AccountSystem** | `Utils/AccountSystem/*` | Pure C++, không dependency Arduino |
| **PageManager** | `Utils/PageManager/*` | Chỉ dùng LVGL + STL (vector, stack) |
| **PingPongBuffer** | `Utils/AccountSystem/PingPongBuffer/*` | Pure C |

### ✅ Application Layer — Giữ 100%

| Module | Files | Lý do |
|---|---|---|
| **App.cpp/h** | `App/App.*` | Chỉ gọi API framework, không hardware |
| **Account_Master** | `App/Accounts/Account_Master.*` | Pure C++ |
| **ACT_Storage** | `App/Accounts/ACT_Storage.cpp` | AccountSystem + StorageService |
| **ACT_Power** | `App/Accounts/ACT_Power.cpp` | AccountSystem + Filters |
| **ACT_IMU** | `App/Accounts/ACT_IMU.cpp` | AccountSystem + HAL_Def |
| **ACT_MusicPlayer** | `App/Accounts/ACT_MusicPlayer.cpp` | AccountSystem + HAL_Audio |
| **ACT_SysConfig** | `App/Accounts/ACT_SysConfig.cpp` | AccountSystem |
| **ACT_Def.h** | `App/Accounts/ACT_Def.h` | Pure struct defines |
| **Tất cả Pages** | `App/Pages/*` | Chỉ dùng LVGL + AccountSystem |
| **ResourcePool** | `App/Resources/*` | LVGL resource management |
| **AppFactory** | `App/Pages/AppFactory.*` | Pure C++ object factory |

### ✅ Utilities — Giữ 100%

| Module | Files | Lý do |
|---|---|---|
| **Filters** | `Utils/Filters/*` | Template headers, pure math |
| **MapConv, TileConv** | `Utils/MapConv/*`, `Utils/TileConv/*` | Pure math + GPS transform |
| **GPX, GPX_Parser** | `Utils/GPX/*`, `Utils/GPX_Parser/*` | File I/O (portable) |
| **TrackFilter** | `Utils/TrackFilter/*` | Pure math |
| **TonePlayer** | `Utils/TonePlayer/*` | Pure C++ + callback |
| **ButtonEvent** | `Utils/ButtonEvent/*` | Pure C++ (cần thay tick source) |
| **Time** | `Utils/Time/*` | Pure C |
| **lv_ext** | `Utils/lv_ext/*` | LVGL extension |
| **lv_allocator** | `Utils/lv_allocator/*` | LVGL memory |
| **3DEngine** | `Utils/3DEngine/*` | Pure math, fixed-point |

### ✅ Giữ có chỉnh sửa nhỏ

| Module | Thay đổi |
|---|---|
| **Config.h** | Cập nhật pin definitions cho ESP32-S3 |
| **Version.h** | Bỏ `#include "SD.h"` (dòng 52) |
| **StorageService** | ArduinoJson → thư viện C++ độc lập (vẫn portable) |
| **ResourceManager** | LVGL giống nhau trên mọi nền tảng |

---

## 3. Những phần phải viết lại

### ❌ Toàn bộ HAL Layer — Viết lại hoàn toàn

#### Bảng mapping Arduino → ESP-IDF

| HAL Module | Arduino API | ESP-IDF API | Mức độ thay đổi |
|---|---|---|---|
| **HAL_Backlight** | `ledcSetup`, `ledcAttachPin`, `ledcWrite` | `ledc_channel_config`, `ledc_set_duty` | ⚡ Thay đổi API |
| **HAL_IMU** | `MPU9250.h` (Arduino lib) + I2C | `i2c_master_read` + tự viết driver MPU6050 | 🔄 Viết lại driver |
| **HAL_Power** | `pinMode`, `digitalWrite`, `analogRead` | `gpio_set_direction`, `adc_oneshot_read` | 🔄 Viết lại hoàn toàn |
| **HAL_Encoder** | `pinMode`, `attachInterrupt`, `digitalRead` | `gpio_install_isr_service`, `gpio_isr_handler_add` | 🔄 Viết lại |
| **HAL_Buzz** | `ledcSetup`, `ledcWriteTone` | `ledc_channel_config`, `ledc_set_freq` | ⚡ Thay đổi API |
| **HAL_Audio** | `TonePlayer` (pure C++ → giữ) + Buzz callback | Giống bên trái | ⚡ Chỉ sửa Buzz call |
| **HAL_SdCard** | `SPI.h`, `SD.h` (Arduino) | `sdmmc_host_t`, `sdspi_host_init` | 🔄 Viết lại hoàn toàn |
| **HAL_Bluetooth** | `NimBLEDevice.h` (Arduino wrapper) | ESP-NimBLE trực tiếp | ⚡ API thay đổi |
| **HAL_I2C_Scan** | `Wire.h` | `i2c_master_scan` | 🔄 Viết lại |
| **HAL_Def.h** | Pure struct → **giữ nguyên** | — | ✅ Giữ |
| **CommonMacro.h** | Pure macros → **giữ nguyên** | — | ✅ Giữ |

### ❌ Port Layer — Viết lại

| Module | Arduino API | ESP-IDF API | Mức độ |
|---|---|---|---|
| **Display.cpp** | TFT_eSPI (Arduino lib) | `spi_device_transmit` + LVGL port | 🔄 Viết lại |
| **lv_port_disp** | Dùng TFT_eSPI `pushColors` | Dùng SPI driver trực tiếp | 🔄 Viết lại |
| **lv_port_indev** | Gọi `HAL::Encoder_GetDiff` | Gọi API tương tự (HAL mới) | ⚡ Sửa nhẹ |
| **lv_port_fatfs** | `SD.h` (Arduino) | `esp_vfs_fat_sdspi_mount` | 🔄 Viết lại |

### ❌ Entry Point — Viết lại

| Thành phần | Arduino | ESP-IDF |
|---|---|---|
| **main** | `void setup()` + `void loop()` | `void app_main()` |
| **Init** | Gọi 3 hàm trong setup() | Gọi 3 hàm trong app_main() |
| **Loop** | loop() gọi HAL::Update() + delay(20) | `while(1) { HAL::Update(); vTaskDelay(20); }` |
| **LVGL task** | `xTaskCreate(TaskLvglUpdate)` | Giống (FreeRTOS) — chỉnh `ulTaskNotifyTake` |

---

## 4. BSP (Board Support Package) cần thay thế

BSP = Board Support Package — code giao tiếp trực tiếp với hardware.

### 4.1 LCD Driver (ST7789)

```
HIỆN TẠI:                           THAY BẰNG:
─────────────────────────            ─────────────────────────
TFT_eSPI (Arduino lib)              ESP-IDF SPI master driver
  ├── screen.begin()                  └── spi_bus_add_device()
  ├── screen.setRotation()
  ├── screen.fillScreen()
  └── screen.pushColors()             └── spi_device_transmit()
```

TFT_eSPI gồm: processor init, SPI transaction, DMA, gamma, rotation.
Trên ESP-IDF, có thể dùng:
1. **Tự viết**: SPI driver + ST7789 init sequence (~200 dòng)
2. **esp_lcd** (ESP-IDF component): `esp_lcd_new_panel_io_spi()`, `esp_lcd_new_panel_st7789()`
3. **LVGL port** trực tiếp qua `esp_lcd` + LVGL `disp_drv`

**Khuyến nghị**: Dùng `esp_lcd` (ESP-IDF built-in ST7789 driver) + LVGL display port.

### 4.2 IMU Driver (MPU6050/MPU9250)

```
HIỆN TẠI:                           THAY BẰNG:
─────────────────────────            ─────────────────────────
MPU9250.h (Arduino lib)             ESP-IDF I2C + tự viết driver
  ├── mpu.setup(0x68)                 └── i2c_master_cmd_begin()
  └── mpu.update()                    └── i2c_write_read() + tính toán
  └── mpu.getAccX() ...               └── đọc register + convert
```

Có thể dùng:
1. **Tự viết**: Đọc MPU6050 registers qua I2C (~150 dòng)
2. **ICM-20948** (nếu dùng S3 + cảm biến mới): ESP-IDF có driver mẫu
3. Thư viện C thuần (port từ Arduino MPU9250)

### 4.3 SD Card

```
HIỆN TẠI:                           THAY BẰNG:
─────────────────────────            ─────────────────────────
SPI.h + SD.h (Arduino)              ESP-IDF sdmmc/sdspi
  ├── SPIClass(HSPI)                 ├── sdmmc_host_t (SDMMC)
  ├── SD.begin(CS, *spi, 80MHz)      ├── sdspi_host_init_device() (SD SPI)
  └── SD.cardType() ...              └── esp_vfs_fat_sdmmc_mount()
```

ESP32-S3 có **SDMMC controller hardware** (không cần SPI cho SD card).
Khuyến nghị: Dùng **SDMMC** (4-bit, tốc độ cao hơn) hoặc **SDSPI**.

### 4.4 Bluetooth

```
HIỆN TẠI:                           THAY BẰNG:
─────────────────────────            ─────────────────────────
NimBLEDevice.h (Arduino wrapper)    ESP-NimBLE (native ESP-IDF)
  ├── NimBLEDevice::init()           ├── nimble_port_init()
  ├── NimBLEScan                     ├── ble_gap_disc()
  ├── NimBLEClient                   └── ble_gap_connect()
  └── NimBLERemoteCharacteristic
```

API thay đổi hoàn toàn. Từ C++ wrapper sang C API của NimBLE.

---

## 5. Driver cần thay thế

### Bảng chi tiết từng driver

| Driver | Arduino | ESP-IDF | File thay thế |
|---|---|---|---|
| **GPIO Output** | `pinMode(pin, OUTPUT)` + `digitalWrite(pin, val)` | `gpio_set_direction(pin, GPIO_MODE_OUTPUT)` + `gpio_set_level(pin, val)` | `driver/gpio.h` |
| **GPIO Input** | `pinMode(pin, INPUT_PULLUP)` + `digitalRead(pin)` | `gpio_set_pull_mode(pin, GPIO_PULLUP_ONLY)` + `gpio_get_level(pin)` | `driver/gpio.h` |
| **GPIO ISR** | `attachInterrupt(pin, handler, CHANGE)` | `gpio_install_isr_service(0)` + `gpio_isr_handler_add(pin, handler, arg)` | `driver/gpio.h` |
| **ADC** | `analogRead(pin)` | `adc_oneshot_read(unit, channel, &value)` | `driver/adc.h` |
| **I2C** | `Wire.begin(SDA, SCL)` | `i2c_master_init()` + `i2c_master_cmd_begin()` | `driver/i2c.h` |
| **SPI** | `SPI.begin(SCK, MISO, MOSI, CS)` | `spi_bus_initialize()` + `spi_bus_add_device()` | `driver/spi_master.h` |
| **PWM** | `ledcSetup(ch, freq, res)` + `ledcAttachPin(pin, ch)` | `ledc_channel_config()` | `driver/ledc.h` |
| **Timer** | `millis()`, `micros()` | `esp_timer_get_time()` | `esp_timer.h` |
| **SD SPI** | `SD.begin(CS, *spi, freq)` | `sdspi_host_init()` + `esp_vfs_fat_sdspi_mount()` | `sdmmc_cmd.h` |
| **SD MMC** | Không hỗ trợ | `sdmmc_host_init()` + `esp_vfs_fat_sdmmc_mount()` | `sdmmc_cmd.h` |
| **BLE** | `NimBLEDevice.h` | `nimble_port_init()` + `ble_gap_disc()` | `nimble/nimble_port.h` |
| **Delay** | `delay(ms)` | `vTaskDelay(pdMS_TO_TICKS(ms))` | `freertos/FreeRTOS.h` |
| **Serial** | `Serial.begin()` + `Serial.print()` | `esp_rom_uart_tx()` hoặc `printf()` | `esp_rom_uart.h` |
| **Log** | `Serial.printf(...)` | `ESP_LOGI(tag, fmt, ...)` | `esp_log.h` |

### Mẫu chuyển đổi GPIO

```cpp
// ─── ARDUINO ───────────────────────────────────────
// HAL_Power.cpp
pinMode(CONFIG_BAT_DET_PIN, INPUT);
pinMode(CONFIG_POWER_EN_PIN, OUTPUT);
digitalWrite(CONFIG_POWER_EN_PIN, LOW);
int val = digitalRead(CONFIG_BAT_CHG_DET_PIN);

// ─── ESP-IDF ───────────────────────────────────────
// hal_power.c (C file mới)
#include "driver/gpio.h"

gpio_set_direction(CONFIG_BAT_DET_PIN, GPIO_MODE_INPUT);
gpio_set_direction(CONFIG_POWER_EN_PIN, GPIO_MODE_OUTPUT);
gpio_set_level(CONFIG_POWER_EN_PIN, 0);
int val = gpio_get_level(CONFIG_BAT_CHG_DET_PIN);
```

### Mẫu chuyển đổi ISR

```cpp
// ─── ARDUINO ───────────────────────────────────────
// HAL_Encoder.cpp
static void Encoder_IrqHandler() {
    uint8_t a = digitalRead(CONFIG_ENCODER_A_PIN);
    // ...
}
attachInterrupt(CONFIG_ENCODER_A_PIN, Encoder_IrqHandler, CHANGE);

// ─── ESP-IDF ───────────────────────────────────────
// hal_encoder.c
#include "driver/gpio.h"

static void IRAM_ATTR encoder_isr_handler(void* arg) {
    uint8_t a = gpio_get_level(CONFIG_ENCODER_A_PIN);
    // ...
}

gpio_install_isr_service(ESP_INTR_FLAG_LEVEL1);
gpio_isr_handler_add(CONFIG_ENCODER_A_PIN, encoder_isr_handler, NULL);
gpio_set_intr_type(CONFIG_ENCODER_A_PIN, GPIO_INTR_ANYEDGE);
```

### Mẫu chuyển đổi I2C

```cpp
// ─── ARDUINO ───────────────────────────────────────
// HAL_I2C_Scan.cpp, HAL_IMU.cpp
Wire.begin(CONFIG_MCU_SDA_PIN, CONFIG_MCU_SCL_PIN);
Wire.beginTransmission(address);
uint8_t error = Wire.endTransmission();

// ─── ESP-IDF ───────────────────────────────────────
// hal_i2c.c
#include "driver/i2c.h"

i2c_config_t conf = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = CONFIG_MCU_SDA_PIN,
    .scl_io_num = CONFIG_MCU_SCL_PIN,
    .master.clk_speed = 400000
};
i2c_param_config(I2C_NUM_0, &conf);
i2c_driver_install(I2C_NUM_0, I2C_MODE_MASTER, 0, 0, 0);

// Scan I2C bus
uint8_t error;
i2c_cmd_handle_t cmd = i2c_cmd_link_create();
i2c_master_start(cmd);
i2c_master_write_byte(cmd, (address << 1) | I2C_MASTER_WRITE, true);
i2c_master_stop(cmd);
error = i2c_master_cmd_begin(I2C_NUM_0, cmd, pdMS_TO_TICKS(100));
i2c_cmd_link_delete(cmd);
```

### Mẫu chuyển đổi SPI (LCD)

```cpp
// ─── ARDUINO (TFT_eSPI) ───────────────────────────
// Display.cpp, lv_port_disp.cpp
SCREEN_CLASS screen;
screen.begin();
screen.pushColors((uint16_t*)color_p, w * h, true);

// ─── ESP-IDF (esp_lcd) ────────────────────────────
// display.c
#include "esp_lcd_panel_io.h"
#include "esp_lcd_panel_st7789.h"
#include "esp_lcd_panel_ops.h"

esp_lcd_panel_io_handle_t io_handle;
esp_lcd_panel_handle_t panel_handle;

esp_lcd_panel_io_spi_config_t io_config = {
    .dc_gpio_num = CONFIG_SCREEN_DC_PIN,
    .cs_gpio_num = CONFIG_SCREEN_CS_PIN,
    .pclk_hz = 80 * 1000 * 1000,
    .spi_mode = 0,
    .trans_queue_depth = 10,
    .lcd_cmd_bits = 8,
    .lcd_param_bits = 8,
};
esp_lcd_new_panel_io_spi(SPI2_HOST, &io_config, &io_handle);
esp_lcd_new_panel_st7789(io_handle, &panel_config, &panel_handle);
esp_lcd_panel_reset(panel_handle);
esp_lcd_panel_init(panel_handle);

// LVGL flush callback dùng esp_lcd
static void disp_flush_cb(lv_disp_drv_t* drv, const lv_area_t* area, lv_color_t* color_p) {
    esp_lcd_panel_draw_bitmap(panel_handle, area->x1, area->y1, area->x2 + 1, area->y2 + 1, color_p);
    lv_disp_flush_ready(drv);
}
```

---

## 6. FreeRTOS Mapping

### 6.1 Task Creation

```
Arduino                          ESP-IDF
─────────────────────────        ─────────────────────────
                                  void app_main() {
void setup() {                        // Không có loopTask
    HAL::Init();                      HAL::Init();
    Port_Init();                      Port_Init();
    App_Init();                       App_Init();
}                                     while(1) {
void loop() {                             HAL::Update();
    HAL::Update();                        vTaskDelay(pdMS_TO_TICKS(20));
    delay(20);                        }
}                                 }
```

### 6.2 Task Changes

| Task | Arduino | ESP-IDF | Viết lại? |
|---|---|---|---|
| **loopTask** | Tự động tạo (Arduino core) | ❌ Không tồn tại | 🔄 Chuyển logic vào `app_main()` |
| **LvglThread** | `xTaskCreate(TaskLvglUpdate, "LvglThread", 20000, ..., MAX-1, &h)` | Giống (FreeRTOS native) | ⚡ Giữ, chỉ sửa include |
| **BuzzerThread** | `xTaskCreate(...)` | Giống | ⚡ Giữ |
| **NimBLE host** | Qua NimBLEDevice C++ wrapper | `nimble_port_freertos_init()` | 🔄 Thay đổi init |
| **NimBLE LL** | Qua NimBLEDevice | `nimble_port_freertos_init()` | 🔄 Thay đổi init |

### 6.3 FreeRTOS API Mapping

| API | Arduino | ESP-IDF |
|---|---|---|
| **Task delay** | `delay(ms)` | `vTaskDelay(pdMS_TO_TICKS(ms))` |
| **Tick** | `millis()` | `pdTICKS_TO_MS(xTaskGetTickCount())` |
| **Task notify** | `xTaskNotifyGive` | ✅ Giống (cùng FreeRTOS) |
| **Task create** | `xTaskCreate` | ✅ Giống |
| **Queue/Sema** | `xQueueCreate` | ✅ Giống |
| **Priority** | `configMAX_PRIORITIES - 1` | ✅ Giống |
| **IRAM func** | `IRAM_ATTR` | ✅ Giống |

### 6.4 Sự khác biệt quan trọng

```
Arduino ESP32:
  FreeRTOS config:
    configMAX_PRIORITIES = 25
    configTICK_RATE_HZ = 1000
    loopTask priority = 1

ESP-IDF (ESP32-S3):
  FreeRTOS config:
    configMAX_PRIORITIES = 56  (SMP)
    configTICK_RATE_HZ = 1000
    configNUM_CORES = 2
    configUSE_CORE_AFFINITY = 1
    configRUN_MULTIPLE_CORES = 1

  ⚠️ Quan trọng: ESP-IDF FreeRTOS là SMP (Symmetric Multi-Processing)
     Arduino ESP32 sử dụng FreeRTOS với 1 core chính + cơ chế khác.
     Trên S3, cần set task affinity bằng xTaskCreatePinnedToCore().
```

---

## 7. LVGL Mapping

LVGL v8.1 trên Arduino và ESP-IDF dùng **cùng API**. Sự khác biệt chỉ ở port layer.

| Thành phần | Arduino | ESP-IDF |
|---|---|---|
| **LVGL init** | `lv_init()` | ✅ Giống |
| **Display driver** | `lv_port_disp_init()` qua TFT_eSPI | 🔄 Dùng `esp_lcd` SPI hoặc tự viết SPI |
| **Input driver** | `lv_port_indev_init()` qua encoder ISR | ⚡ Giống (chỉ đổi HAL API) |
| **FS driver** | `lv_fs_if_init()` qua SD + FatFS | 🔄 Dùng `esp_vfs_fat` mount → LVGL |
| **Tick** | `LV_TICK_CUSTOM = 1`, dùng `millis()` | ⚡ `LV_TICK_CUSTOM = 1`, dùng `esp_timer_get_time()` |
| **Timer** | `lv_timer_create()` | ✅ Giống |
| **Animation** | `lv_anim_start()` | ✅ Giống |
| **Style** | `lv_style_set_*()` | ✅ Giống |
| **Display buffer** | `malloc` buffer 240×120 | ⚡ Có thể malloc 240×240×2 (full screen) nhờ PSRAM |
| **DMA** | Không dùng | ✅ Có thể dùng `esp_lcd` với DMA |

### LVGL Display Buffer (ESP32-S3 lợi thế)

```
HIỆN TẠI (ESP32-PICO, 520KB SRAM):
  DISP_BUF_SIZE = 240 × 120 = 28,800 pixel
  Buffer size: 28,800 × 2 = 57.6 KB
  Type: Single buffer

MỤC TIÊU (ESP32-S3 + 2MB PSRAM):
  DISP_BUF_SIZE = 240 × 240 = 57,600 pixel
  Buffer size: 57,600 × 2 = 115.2 KB
  Type: Double buffer (LVGL dual buffer với DMA)
  → Không còn tearing, full 60 FPS
```

---

## 8. Kiến trúc thư mục đề xuất

```
Peak-ESP32-S3/
├── CMakeLists.txt                    ← Root CMake (dự án ESP-IDF)
├── sdkconfig                         ← ESP-IDF config (idf.py menuconfig)
├── sdkconfig.defaults                ← Default config
│
├── main/
│   ├── CMakeLists.txt                ← Component CMake
│   ├── app_main.cpp                  ← app_main() entry (thay main.cpp)
│   │
│   ├── hal/                          ← ESP-IDF HAL (VIẾT LẠI)
│   │   ├── hal_backlight.c
│   │   ├── hal_power.c
│   │   ├── hal_encoder.c
│   │   ├── hal_buzz.c
│   │   ├── hal_audio.c
│   │   ├── hal_sdcard.c
│   │   ├── hal_imu.c
│   │   ├── hal_bluetooth.c
│   │   ├── hal_i2c.c
│   │   ├── hal_def.h                  ← GIỮ (structs)
│   │   ├── hal.h                      ← VIẾT LẠI (API namespace)
│   │   └── common_macro.h             ← GIỮ
│   │
│   ├── port/                          ← VIẾT LẠI (ESP-IDF + esp_lcd)
│   │   ├── display.c
│   │   ├── lv_port_disp.c
│   │   ├── lv_port_indev.c
│   │   └── lv_port_fatfs.c
│   │
│   ├── app/                           ← GIỮ NGUYÊN (trừ App.h)
│   │   ├── CMakeLists.txt
│   │   ├── app.h / app.cpp
│   │   │
│   │   ├── accounts/
│   │   │   ├── account_master.h/.cpp
│   │   │   ├── act_def.h
│   │   │   ├── act_list.inc
│   │   │   ├── act_storage.cpp
│   │   │   ├── act_power.cpp
│   │   │   ├── act_imu.cpp
│   │   │   ├── act_music_player.cpp   ← renamed from ACT_MusicPlayer
│   │   │   └── act_sys_config.cpp
│   │   │
│   │   ├── pages/
│   │   │   ├── app_factory.h/.cpp
│   │   │   ├── page.h                 ← include path fix
│   │   │   ├── startup/
│   │   │   ├── template/
│   │   │   ├── system_infos/
│   │   │   └── status_bar/
│   │   │
│   │   ├── resources/
│   │   │   ├── resource_pool.h/.cpp
│   │   │   ├── font/
│   │   │   └── image/
│   │   │
│   │   └── configs/
│   │       ├── config.h               ← SỬA pin definitions
│   │       └── version.h              ← SỬA (bỏ SD.h include)
│   │
│   └── utils/                         ← GIỮ NGUYÊN
│       ├── account_system/
│       │   ├── account.h/.cpp
│       │   ├── account_broker.h/.cpp
│       │   └── ping_pong_buffer/
│       ├── page_manager/
│       ├── filters/
│       ├── map_conv/
│       │   ├── tile_system/
│       │   └── gps_transform/
│       ├── tile_conv/
│       ├── gpx/
│       ├── gpx_parser/
│       ├── track_filter/
│       ├── tone_player/
│       ├── button_event/
│       ├── time/
│       ├── lv_ext/
│       ├── lv_allocator/
│       └── 3d_engine/
│
├── components/                        ← ESP-IDF components
│   ├── lvgl/                          ← LVGL (ESP-IDF component)
│   │   └── CMakeLists.txt + sources
│   ├── arduino_json/                  ← ArduinoJson (portable)
│   └── mpu6050/                       ← Tự viết driver MPU6050
│
└── partitions.csv                     ← Partition table
```

### File CMakeLists.txt mẫu

```cmake
# main/CMakeLists.txt
idf_component_register(
    SRCS
        app_main.cpp
        # HAL
        hal/hal_backlight.c
        hal/hal_power.c
        hal/hal_encoder.c
        hal/hal_buzz.c
        hal/hal_audio.c
        hal/hal_sdcard.c
        hal/hal_imu.c
        hal/hal_bluetooth.c
        hal/hal_i2c.c
        # Port
        port/display.c
        port/lv_port_disp.c
        port/lv_port_indev.c
        port/lv_port_fatfs.c
        # App (keep)
        app/app.cpp
        app/accounts/account_master.cpp
        app/accounts/act_storage.cpp
        app/accounts/act_power.cpp
        app/accounts/act_imu.cpp
        app/accounts/act_music_player.cpp
        app/accounts/act_sys_config.cpp
        app/pages/app_factory.cpp
        app/pages/startup/startup.cpp
        app/pages/startup/startup_view.cpp
        app/pages/startup/startup_model.cpp
        app/pages/template/template.cpp
        app/pages/template/template_view.cpp
        app/pages/template/template_model.cpp
        app/pages/system_infos/system_infos.cpp
        app/pages/system_infos/system_infos_view.cpp
        app/pages/system_infos/system_infos_model.cpp
        app/pages/status_bar/status_bar.cpp
        app/resources/resource_pool.cpp
        # Utils (keep)
        utils/account_system/account.cpp
        utils/account_system/account_broker.cpp
        utils/account_system/ping_pong_buffer/ping_pong_buffer.c
        utils/page_manager/pm_base.cpp
        utils/page_manager/pm_router.cpp
        utils/page_manager/pm_state.cpp
        utils/page_manager/pm_anim.cpp
        utils/page_manager/pm_drag.cpp
        utils/page_manager/resource_manager.cpp
        utils/tone_player/tone_player.cpp
        utils/button_event/button_event.cpp
        utils/gpx/gpx.cpp
        utils/gpx_parser/gpx_parser.cpp
        utils/track_filter/track_point_filter.cpp
        utils/track_filter/track_line_filter.cpp
        utils/map_conv/map_conv.cpp
        utils/map_conv/tile_system/tile_system.cpp
        utils/map_conv/gps_transform/gps_transform.c
        utils/tile_conv/tile_conv.cpp
        utils/time/time.cpp
        utils/time/date_strings.cpp
        utils/lv_ext/lv_anim_timeline_wrapper.c
        utils/lv_ext/lv_label_anim_effect.cpp
        utils/lv_ext/lv_obj_ext_func.cpp
        # StorageService
        utils/storage_service/storage_service.cpp
    INCLUDE_DIRS
        .
        hal
        port
        app
        app/configs
        app/accounts
        app/pages
        app/resources
        utils
        utils/account_system
        utils/page_manager
        utils/filters
        utils/3d_engine
        utils/lv_ext
        utils/lv_allocator
        utils/storage_service
    REQUIRES
        lvgl
        driver
        esp_lcd
        sdmmc
        fatfs
        nvs_flash
        nimble
        arduino_json
        mpu6050
)
```

---

## 9. Sơ đồ thay đổi

```
Peak (Arduino)                        Peak-ESP32-S3 (ESP-IDF)
═════════════════                     ═══════════════════════

src/main.cpp                          main/app_main.cpp
  setup() → 3 init calls                app_main() → 3 init calls
  loop()                                while(1)

src/HAL/*.cpp                          main/hal/*.{c,h}     ← VIẾT LẠI
  Arduino: pinMode, digitalRead,        ESP-IDF: gpio_* API
           analogRead, attachInterrupt  adc_*, i2c_*, ledc_*
           Wire, SPI, SD, ledc

src/Port/Display.cpp                   main/port/display.c   ← VIẾT LẠI
  TFT_eSPI.begin()                      esp_lcd_new_panel_*
  xTaskCreate(LvglThread)               xTaskCreate(LvglThread) ✅

src/Port/lv_port/*.cpp                 main/port/lv_port_*   ← VIẾT LẠI
  TFT_eSPI pushColors()                 esp_lcd_panel_draw_bitmap()

src/App/App.h                          main/app/app.h        ← SỬA
  INIT_DONE()→xTaskNotifyGive           Giữ (FreeRTOS giống)

src/App/Configs/Config.h               main/app/configs/config.h ← SỬA
  Pin definitions ESP32-PICO            Pin definitions ESP32-S3

src/App/** (trừ HAL/Port)              main/app/**           ✅ GIỮ
  AccountSystem, PageManager,           Chỉ sửa include paths
  Pages, Resources, Utils

lib/TFT_eSPI/                          components/lvgl/      ✅ THAY
lib/ArduinoJson/                       components/arduino_json/
lib/NimBLE/                            components (ESP-NimBLE built-in)

──────────────────────────────────────────────────────────
                    TỔNG KẾT
──────────────────────────────────────────────────────────
Files giữ nguyên:        ~80%  (~60 files)
Files viết lại/toàn bộ:  ~15%  (~12 files: HAL, Port, entry)
Files sửa nhẹ:           ~5%   (~4 files: Config, App, Version)
```

---

## 10. Kế hoạch triển khai

### Phase 1: Setup project (1 ngày)
1. Tạo ESP-IDF project với `idf.py create-project`
2. Cấu hình `sdkconfig` (PSRAM, flash, SPI, I2C, NimBLE)
3. Thêm LVGL component (`idf_component.yml` hoặc git submodule)
4. Copy toàn bộ `app/` và `utils/` từ Peak
5. Cấu hình CMakeLists.txt

### Phase 2: BSP + HAL (2-3 ngày)
1. Viết LCD driver (`esp_lcd` ST7789)
2. Viết GPIO driver (Backlight, Encoder, Power)
3. Viết I2C driver + MPU6050
4. Viết SDMMC/SDSPI driver
5. Viết LEDC driver (Buzzer, Backlight)
6. Viết ADC driver (Battery)
7. Viết init sequence trong `HAL::Init()`

### Phase 3: Port Layer (1 ngày)
1. Viết `lv_port_disp` dùng `esp_lcd`
2. Viết `lv_port_indev` dùng encoder (HAL mới)
3. Viết `lv_port_fatfs` dùng `esp_vfs_fat`
4. Viết `app_main.cpp` entry point

### Phase 4: App layer (0.5 ngày)
1. Sửa `Config.h` (pin definitions cho ESP32-S3)
2. Sửa `App.h` (include paths)
3. Sửa `Version.h` (bỏ `SD.h`)
4. Build + fix lỗi include

### Phase 5: Test (1 ngày)
1. Boot test: UART log
2. LCD + LVGL test
3. Encoder + input test
4. IMU I2C test
5. SD card test
6. BLE connection test
7. Power management test

---

## 11. Tổng kết

| Hạng mục | Chi tiết |
|---|---|
| **Giữ nguyên** | AccountSystem, PageManager, Filters, Map/Tile/GPX/Track, TonePlayer, ButtonEvent, StorageService, Time, Pages, Resources, 3DEngine, CommonMacro (~80% code) |
| **Viết lại** | HAL layer (9 modules: backlight, power, encoder, buzz, audio, sd, imu, bt, i2c), Display port, lv_port_disp, lv_port_fatfs (~15%) |
| **Sửa nhẹ** | Config.h (pins), Version.h (SD include), App.h (INIT_DONE), app_main thay setup/loop (~5%) |
| **BSP thay thế** | TFT_eSPI → `esp_lcd`, Arduino SD → `esp_vfs_fat_sdmmc` |
| **Driver thay thế** | Arduino GPIO/I2C/SPI/ADC/PWM → ESP-IDF `driver/*.h` |
| **FreeRTOS** | Tương thích (cùng nhân), chỉ khác entry point |
| **LVGL** | Cùng phiên bản, chỉ khác port layer |
| **Lợi thế S3** | PSRAM → full screen double buffer, SDMMC hardware, CPU mạnh hơn |
