# 🔷 Peak Project — Complete Module & Dependency Map

> Liệt kê toàn bộ module trong firmware Peak, với chức năng, public API, dependency và dependents.

---

## 1. Entry Point & Config

### `main.cpp`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Entry point của firmware. Gọi `setup()` 1 lần, `loop()` vô hạn. |
| **File** | `src/main.cpp` |
| **Public API** | `setup()`, `loop()` (Arduino entry) |
| **Phụ thuộc** | `HAL/HAL.h`, `Port/Display.h`, `App/App.h` |
| **Sử dụng bởi** | Arduino core (`loopTask`) |

### `Configs/Config.h`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Định nghĩa toàn bộ hằng số cấu hình: pin GPIO, kích thước màn hình, GPS default, đường dẫn file, v.v. |
| **File** | `src/App/Configs/Config.h` |
| **Public API** | `#define CONFIG_*` (pin, kích thước, timeout, v.v.) |
| **Phụ thuộc** | Không |
| **Sử dụng bởi** | Mọi module (HAL, Port, App, Accounts, Pages) |

### `Configs/Version.h`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Định nghĩa version: firmware name (`littleVisual`), version số, tên tác giả (`PZH`), LVGL version, compiler version, build time |
| **File** | `src/App/Configs/Version.h` |
| **Public API** | `VERSION_FIRMWARE_NAME`, `VERSION_SOFTWARE`, `VERSION_HARDWARE`, `VERSION_AUTHOR_NAME`, `VERSION_LVGL`, `VERSION_COMPILER`, `VERSION_BUILD_TIME` |
| **Phụ thuộc** | `lvgl.h` (lấy LVGL_VERSION) |
| **Sử dụng bởi** | `HAL.cpp`, `SystemInfos` page |

### `CommonMacro.h`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Thư viện macro tiện ích: `__IntervalExecute`, `__LimitValue`, `__ValueMonitor`, `__ExecuteOnce`, `__SemaphoreTake`, `__Map`, v.v. |
| **File** | `src/HAL/CommonMacro.h` |
| **Public API** | `__IntervalExecute(func,time)`, `__LimitValue(x,min,max)`, `__Map(x,in_min,in_max,out_min,out_max)`, `__ExecuteOnce(func)`, v.v. |
| **Phụ thuộc** | Không (chỉ macro thuần) |
| **Sử dụng bởi** | `HAL.cpp`, `HAL_Power.cpp`, `ACT_Power.cpp`, và nhiều module khác |

---

## 2. HAL Layer (Hardware Abstraction)

### `HAL` (namespace tổng)

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | namespace chứa tất cả API HAL. File `HAL.cpp` điều phối Init/Update tuần tự. |
| **File** | `src/HAL/HAL.h`, `src/HAL/HAL.cpp`, `src/HAL/HAL_Def.h` |
| **Public API** | `HAL::Init()`, `HAL::Update()` |
| **Phụ thuộc** | Tất cả module HAL con, `Config.h`, `CommonMacro.h`, `FreeRTOS.h` |
| **Sử dụng bởi** | `main.cpp` (gọi Init/Update) |

### `HAL::Backlight`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Điều khiển đèn nền LCD qua PWM (LEDC), hỗ trợ gradual fade qua LVGL animation |
| **File** | `src/HAL/HAL_Backlight.cpp` |
| **Public API** | `Backlight_Init()`, `Backlight_GetValue()`, `Backlight_SetValue(val)`, `Backlight_SetGradual(target,time)`, `Backlight_ForceLit(en)` |
| **Phụ thuộc** | `Config.h` (pin BLK), `ledc` (ESP32), `lv_anim` |
| **Sử dụng bởi** | `HAL::Init()`, `Port_Init()`, `HAL_Power::Shutdown()` |

### `HAL::IMU` (MPU6050/9250)

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Đọc cảm biến IMU 9-axis: accelerometer, gyroscope, magnetometer, attitude (roll/pitch/yaw). Commit dữ liệu vào Account System. |
| **File** | `src/HAL/HAL_IMU.cpp` |
| **Public API** | `IMU_Init()`, `IMU_Update()`, `IMU_Info_t` (ax,ay,az,gx,gy,gz,mx,my,mz,roll,yaw,pitch) |
| **Phụ thuộc** | `MPU9250` library (I2C), `Account_Master.h` (để Commit), `Config.h` (I2C pins) |
| **Sử dụng bởi** | `HAL::Init()`, `HAL::Update()`, `AccountSystem::IMU_Commit()` |

### `HAL::Power`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Quản lý nguồn: bật/tắt nguồn chính (qua EN pin), đọc ADC pin, phát hiện sạc, auto low-power shutdown |
| **File** | `src/HAL/HAL_Power.cpp` |
| **Public API** | `Power_Init()`, `Power_Update()`, `Power_HandleTimeUpdate()`, `Power_SetAutoLowPowerTimeout(sec)`, `Power_GetAutoLowPowerTimeout()`, `Power_SetAutoLowPowerEnable(en)`, `Power_Shutdown()`, `Power_GetInfo(info)`, `Power_EventMonitor()` |
| **Phụ thuộc** | `Config.h` (pin BAT_DET, CHG_DET, POWER_EN), `HAL_Backlight` (shutdown fade) |
| **Sử dụng bởi** | `HAL::Init()`, `HAL::Update()`, `ACT_Power` (pull), `HAL_Encoder` (long press shutdown) |

### `HAL::Encoder`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Đọc rotary encoder (A/B + push button). Xử lý interrupt, tính diff, phát hiện short/long press qua ButtonEvent. Tạo Account "Encoder" riêng. |
| **File** | `src/HAL/HAL_Encoder.cpp` |
| **Public API** | `Encoder_Init()`, `Encoder_Update()`, `Encoder_GetDiff()`, `Encoder_GetIsPush()`, `Encoder_SetEnable(en)` |
| **Phụ thuộc** | `Config.h` (ENCODER_A/B/PUSH pins), `ButtonEvent`, `AccountSystem`, `HAL_Buzz` (tone feedback), `HAL_Power` (shutdown) |
| **Sử dụng bởi** | `HAL::Init()`, `HAL::Update()`, `lv_port_indev` (encoder_read), `StartUpModel` (SetEnable) |

### `HAL::Buzzer`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Điều khiển buzzer PWM. Tạo FreeRTOS thread riêng (`BuzzerThread`) để phát tone không-blocking. |
| **File** | `src/HAL/HAL_Buzz.cpp` |
| **Public API** | `Buzz_init()`, `Buzz_SetEnable(en)`, `Buzz_Tone(freq,duration)` |
| **Phụ thuộc** | `Config.h` (pin BUZZ, channel), `ledc` (ESP32), FreeRTOS (`xTaskCreate`) |
| **Sử dụng bởi** | `HAL::Init()`, `HAL_Audio` (callback), `ACT_SysConfig` (enable), `HAL_Encoder` (feedback) |

### `HAL::Audio`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Play nhạc nền (melody) qua TonePlayer. Map tên bài hát → mã nhạc (MusicCode). |
| **File** | `src/HAL/HAL_Audio.cpp` |
| **Public API** | `Audio_Init()`, `Audio_Update()`, `Audio_PlayMusic(name)` |
| **Phụ thuộc** | `TonePlayer` (utils), `MusicCode.h`, `HAL_Buzz` (Tone_Callback) |
| **Sử dụng bởi** | `HAL::Init()` (play "Startup"), `ACT_MusicPlayer`, `HAL_Encoder` (shutdown sound) |

### `HAL::SD_Card`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Khởi tạo và quản lý SD card qua SPI (HSPI riêng, 80MHz). Phát hiện insert/remove. |
| **File** | `src/HAL/HAL_SdCard.cpp` |
| **Public API** | `SD_Init()`, `SD_Update()`, `SD_GetReady()`, `SD_GetCardSizeMB()`, `SD_SetEventCallback(callback)` |
| **Phụ thuộc** | `Config.h` (SD pins, SPI bus), `SPI.h`, `SD.h` (ESP32 SD library) |
| **Sử dụng bởi** | `HAL::Init()`, `HAL::Update()`, `ACT_Storage` (pull SD info), `lv_port_fatfs` |

### `HAL::Bluetooth`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Kết nối BLE với thiết bị ngoài (Dummy-Robot) qua NimBLE stack. Quét và kết nối tự động. |
| **File** | `src/HAL/HAL_Bluetooth.cpp` |
| **Public API** | `BT_Init()`, `BT_Update()`, `BluetoothConnected()` |
| **Phụ thuộc** | `NimBLEDevice.h` (NimBLE library), `Config.h` |
| **Sử dụng bởi** | `HAL::Init()` (gọi sớm nhất), `HAL::Update()`, `StatusBar` (hiển thị icon) |

### `HAL::I2C`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Khởi tạo I2C bus và scan các device trên bus (debug). |
| **File** | `src/HAL/HAL_I2C_Scan.cpp` |
| **Public API** | `I2C_Init(bool startScan)` |
| **Phụ thuộc** | `Wire.h`, `Config.h` (SDA/SCL pins) |
| **Sử dụng bởi** | `HAL::Init()`, `HAL_IMU` (dùng chung Wire) |

### `HAL_Def.h`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Định nghĩa các struct dữ liệu dùng chung trong HAL layer. |
| **File** | `src/HAL/HAL_Def.h` |
| **Public API** | `HAL::Clock_Info_t`, `HAL::IMU_Info_t`, `HAL::Power_Info_t` |
| **Phụ thuộc** | `<stdint.h>` |
| **Sử dụng bởi** | `HAL.h`, `ACT_Def.h`, `Account_Master.h`, `SystemInfosModel`, Pages |

---

## 3. Port Layer

### `Port_Init` / `Display`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Khởi tạo màn hình TFT (ST7789), LVGL core, các port của LVGL, và tạo FreeRTOS task `LvglThread` cho rendering. |
| **File** | `src/Port/Display.cpp`, `src/Port/Display.h` |
| **Public API** | `Port_Init()`, `extern TaskHandle_t handleTaskLvgl`, `SCREEN_CLASS` (TFT_eSPI typedef) |
| **Phụ thuộc** | `TFT_eSPI`, `lvgl.h`, `Config.h`, `HAL_Backlight`, FreeRTOS `xTaskCreate` |
| **Sử dụng bởi** | `main.cpp`, `App.h` (INIT_DONE macro) |

### `lv_port_disp`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Port LVGL cho display: đăng ký flush callback `disp_flush_cb`, khởi tạo draw buffer (single buffer, half screen 240×120). |
| **File** | `src/Port/lv_port/lv_port_disp.cpp` |
| **Public API** | `lv_port_disp_init(screen)`, `disp_flush_cb`, `lv_disp_buf_p` (extern) |
| **Phụ thuộc** | `lvgl.h`, `Port/Display.h`, `Config.h` |
| **Sử dụng bởi** | `Port_Init()`, `HAL::Init()` (malloc buffer) |

### `lv_port_indev`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Port LVGL cho input device (encoder). Đọc encoder diff và push state từ HAL. |
| **File** | `src/Port/lv_port/lv_port_indev.cpp` |
| **Public API** | `lv_port_indev_init()`, `encoder_read()` |
| **Phụ thuộc** | `lvgl.h`, `HAL_Encoder` |
| **Sử dụng bởi** | `Port_Init()` |

### `lv_port_fatfs`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Port LVGL cho file system (FatFS). Đăng ký driver letter `S:` cho SD card. |
| **File** | `src/Port/lv_port/lv_port_fatfs.cpp` |
| **Public API** | `lv_fs_if_init()` |
| **Phụ thuộc** | `ff.h` (FatFS), `lvgl.h`, `SD.h` |
| **Sử dụng bởi** | `Port_Init()`, các module đọc file (Storage, Map, GPX) |

---

## 4. Framework Core

### `AccountSystem`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Pub/Sub Message Middleware — xương sống giao tiếp nội bộ. Cho phép các module đăng ký, subscribe, commit, publish, pull, notify dữ liệu. |
| **Files** | |
| | `src/App/Utils/AccountSystem/Account.h/.cpp` — Class Account: node duy nhất (có thể là publisher lẫn subscriber) |
| | `src/App/Utils/AccountSystem/AccountBroker.h/.cpp` — AccountBroker: registry center, quản lý tất cả Account |
| | `src/App/Utils/AccountSystem/PingPongBuffer/` — Double-buffer cho dữ liệu cache |
| | `src/App/Utils/AccountSystem/AccountSystemLog.h` — Log macro |
| **Public API** | `Account(id, center, bufSize, userData)`, `Subscribe(pubID)`, `Unsubscribe(pubID)`, `Commit(data,size)`, `Publish()`, `Pull(pubID, data, size)`, `Notify(pubID, data, size)`, `SetEventCallback(callback)`, `SetTimerPeriod(period)`, `SetTimerEnable(en)` |
| | `AccountBroker(name)`, `AddAccount`, `RemoveAccount`, `SearchAccount(id)` |
| **Phụ thuộc** | `lvgl.h` (timer), `<vector>`, `<cstring>` |
| **Sử dụng bởi** | **Mọi module trong App layer**: Account nodes, Pages, HAL_Encoder, HAL_IMU, StatusBar |

### `PageManager`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Quản lý vòng đời page theo cơ chế stack (Push/Pop) với animation, lấy cảm hứng từ iOS ViewController. Hỗ trợ cache, stash params, drag. |
| **Files** | |
| | `src/App/Utils/PageManager/PageBase.h` — Base class cho mọi page (8 states, lifecycle callbacks) |
| | `src/App/Utils/PageManager/PageFactory.h` — Abstract factory cho page |
| | `src/App/Utils/PageManager/PageManager.h` — Class PageManager chính |
| | `src/App/Utils/PageManager/PM_Base.cpp` — Install, Register, Uninstall, page pool management |
| | `src/App/Utils/PageManager/PM_Router.cpp` — Push, Pop, SwitchTo, BackHome, SwitchAnimTypeUpdate |
| | `src/App/Utils/PageManager/PM_State.cpp` — State machine: LOAD→WILL_APPEAR→DID_APPEAR→...→UNLOAD |
| | `src/App/Utils/PageManager/PM_Anim.cpp` — Animation type definitions |
| | `src/App/Utils/PageManager/PM_Drag.cpp` — Root drag (swipe) support |
| | `src/App/Utils/PageManager/PM_Log.h` — Log macro |
| | `src/App/Utils/PageManager/ResourceManager.h/.cpp` — Resource caching (dùng cho fonts, images) |
| **Public API** | `PageManager(factory)`, `Install(className,appName)`, `Uninstall(appName)`, `Register(base,name)`, `Unregister(name)` |
| | `Push(name, stash)`, `Pop()`, `BackHome()` |
| | `SetGlobalLoadAnimType(anim, time, path)`, các enum animation (OVER_LEFT, MOVE_TOP, FADE_ON, NONE...) |
| | `PageBase`: `onCustomAttrConfig()`, `onViewLoad()`, `onViewWillAppear()`, `onViewDidAppear()`, `onViewWillDisappear()`, `onViewDidDisappear()`, `onViewDidUnload()`, `SetCustomCacheEnable(en)`, `SetCustomLoadAnimType(...)` |
| **Phụ thuộc** | `lvgl.h`, `<vector>`, `<stack>`, `PageFactory.h` |
| **Sử dụng bởi** | `App.cpp` (sử dụng trực tiếp), tất cả Pages (kế thừa PageBase) |

---

## 5. Framework Utilities

### `Filters`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Thư viện filter số cho dữ liệu cảm biến: Median, Lowpass, Hysteresis, Sliding window, MedianQueue. Template-based. |
| **Files** | `src/App/Utils/Filters/FilterBase.h`, `Filters.h`, `HysteresisFilter.h`, `LowpassFilter.h`, `MedianFilter.h`, `MedianQueueFilter.h`, `SlidingFilter.h` |
| **Public API** | `Filter::Median<T,size>`, `Filter::Lowpass<T>`, `Filter::Hysteresis<T>(threshold)`, `Filter::MedianQueue<T,size>`, `Filter::Sliding<T,size>` — mỗi class có `GetNext(input)` |
| **Phụ thuộc** | `<stdint.h>` |
| **Sử dụng bởi** | `ACT_Power.cpp` (Hysteresis + MedianQueue cho battery) |

### `MapConv`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Chuyển đổi tọa độ GPS ↔ map tile (Bing Maps Tile System). Hỗ trợ WGS84/GCJ-02 transform, level range, quản lý tile directory path. |
| **Files** | `src/App/Utils/MapConv/MapConv.h/.cpp`, `src/App/Utils/MapConv/TileSystem/TileSystem.h/.cpp`, `src/App/Utils/MapConv/GPS_Transform/GPS_Transform.h/.c` |
| **Public API** | `MapConv()`, `SetLevel(level)`, `GetLevel()`, `SetDirPath(path)`, `SetCoordTransformEnable(en)`, `SetLevelRange(min,max)`, `GetMapTile(lon,lat,tile)`, `ConvertPosToTile(x,y,tile)`, `ConvertMapPath(x,y,path,len)`, `ConvertMapCoordinate(lon,lat,mapX,mapY)` |
| **Phụ thuộc** | `lvgl.h` (lv_fs), `math.h` |
| **Sử dụng bởi** | `ACT_Storage.cpp` (gọi API để cấu hình map) |

### `TileConv`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Chuyển đổi tọa độ tile map sang viewport. Quản lý tile container, focus point, offset. |
| **Files** | `src/App/Utils/TileConv/TileConv.h/.cpp` |
| **Public API** | `TileConv(viewWidth,viewHeight,tileSize)`, `SetTileSize(size)`, `SetViewSize(w,h)`, `SetFocusPos(x,y)`, `GetFocusOffset(offset)`, `GetTileContainer(rect)`, `GetTileContainerOffset(offset)`, `GetTilePos(index,pos)`, `FixTile(x,up)`, `GetOffset(offset,point)` |
| **Phụ thuộc** | `<stdint.h>` |
| **Sử dụng bởi** | Module map (chưa tích hợp trong codebase hiện tại) |

### `GPX Parser`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Parse file GPX (GPS Exchange Format) — đọc trackpoints, waypoints, routes từ XML. |
| **Files** | `src/App/Utils/GPX_Parser/GPX_Parser.h/.cpp` |
| **Public API** | `GPX_Parser` (API đọc/parse file GPX) |
| **Phụ thuộc** | File I/O (LVGL fs hoặc stdio) |
| **Sử dụng bởi** | Module track/GPS (chưa tích hợp active) |

### `GPX`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Module xuất dữ liệu track ra file GPX. |
| **Files** | `src/App/Utils/GPX/GPX.h/.cpp` |
| **Public API** | `GPX` (API ghi file GPX) |
| **Phụ thuộc** | File I/O |
| **Sử dụng bởi** | Module track (chưa tích hợp active) |

### `TrackFilter`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Lọc điểm track: `TrackPointFilter` (Douglas-Peucker algorithm) và `TrackLineFilter` (filter đường đi). |
| **Files** | `src/App/Utils/TrackFilter/TrackFilter.h`, `TrackPointFilter.h/.cpp`, `TrackLineFilter.h/.cpp` |
| **Public API** | `TrackPointFilter::pushPoint(point)`, `TrackPointFilter::getFilteredPoints()` (DP simplify), `TrackLineFilter` |
| **Phụ thuộc** | `math.h` |
| **Sử dụng bởi** | Module track/GPS (chưa tích hợp active) |

### `TonePlayer`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Phát nhạc nền (melody) dạng non-blocking. Map các nốt nhạc (C4, D4...) → tần số. |
| **Files** | `src/App/Utils/TonePlayer/TonePlayer.h/.cpp`, `MusicCode.h`, `ToneMap.h` |
| **Public API** | `TonePlayer()`, `SetCallback(callback)`, `Play(musicCode, length)`, `Update(tick)` |
| **Phụ thuộc** | Callback function (do HAL cung cấp) |
| **Sử dụng bởi** | `HAL_Audio.cpp` |

### `ButtonEvent`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Non-blocking button event driver: phát hiện pressed, released, short click, long press, double click, long press repeat. |
| **Files** | `src/App/Utils/ButtonEvent/ButtonEvent.h/.cpp`, `EventType.inc` |
| **Public API** | `ButtonEvent(longPressTime,repeatTime,doubleClickTime)`, `EventAttach(callback)`, `EventMonitor(isPress)`, `GetClicked()`, `GetPressed()`, `GetLongPressed()`, `GetClickCnt()` |
| **Phụ thuộc** | `<stdint.h>`, hệ thống tick (millis) |
| **Sử dụng bởi** | `HAL_Encoder.cpp` (xử lý push button) |

### `StorageService`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Quản lý lưu trữ JSON config trên SD card. Add/remove key-value, save/load file. |
| **Files** | `src/App/Utils/StorageService/StorageService.h/.cpp` |
| **Public API** | `StorageService(filepath)`, `Add(key,value,size,type)`, `Remove(key)`, `SaveFile()`, `LoadFile()` |
| **Phụ thuộc** | `ArduinoJson` library, `<vector>` |
| **Sử dụng bởi** | `ACT_Storage.cpp` |

### `Time`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Thư viện thời gian (TimeLib port). Cung cấp hàm đọc/viết thời gian, date strings. |
| **Files** | `src/App/Utils/Time/Time.h/.cpp`, `TimeLib.h`, `DateStrings.cpp` |
| **Public API** | `hour()`, `minute()`, `second()`, `day()`, `month()`, `year()`, `setTime(t)`, `adjustTime(t)`, v.v. |
| **Phụ thuộc** | Không (pure C) |
| **Sử dụng bởi** | Module clock/GPS (dự kiến) |

### `lv_ext`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Extension cho LVGL: `lv_anim_timeline_wrapper` (dễ tạo animation timeline), `lv_label_anim_effect` (hiệu ứng label), `lv_obj_ext_func` (hàm tiện ích cho object). |
| **Files** | `src/App/Utils/lv_ext/lv_anim_timeline_wrapper.h/.c`, `lv_label_anim_effect.h/.cpp`, `lv_obj_ext_func.h/.cpp` |
| **Public API** | `lv_anim_timeline_add_wrapper(timeline, wrapper)`, `lv_label_anim_effect`, `lv_obj_ext_func` |
| **Phụ thuộc** | `lvgl.h` |
| **Sử dụng bởi** | `StartUpView.cpp` (anim timeline), các page khác |

### `lv_allocator`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Custom memory allocator cho LVGL heap (tùy chọn). |
| **Files** | `src/App/Utils/lv_allocator/lv_allocator.h` |
| **Public API** | Tùy chọn (có thể override malloc/free cho LVGL) |
| **Phụ thuộc** | `lvgl.h` |
| **Sử dụng bởi** | LVGL (nếu được cấu hình) |

### `3DEngine`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Software 3D renderer nhúng dùng fixed-point arithmetic (Q14.14). Có mesh cube, vector, matrix 4x4. Đang phát triển. |
| **Files** | `src/App/Utils/3DEngine/type.h`, `mesh_cube.h` |
| **Public API** | `Vector3 {x,y,z}`, `Matrix4 {m[4][4]}`, `pMultiply(x,y)` (fixed-point multiply) — mesh data: 8 nodes, 12 triangles |
| **Phụ thuộc** | `stdint.h`, `math.h` |
| **Sử dụng bởi** | Scene3D page (chưa tích hợp — `SystemInfos` có reference `Push("Pages/Scene3D")`) |

---

## 6. APP Layer

### `App` (Application Controller)

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Khởi tạo toàn bộ application: Accounts, Resources, Pages, StatusBar. Gửi INIT_DONE signal. |
| **Files** | `src/App/App.h`, `src/App/App.cpp` |
| **Public API** | `App_Init()`, `App_UnInit()`, macro `ACCOUNT_SEND_NOTIFY_CMD(ACT,CMD)`, `INIT_DONE()` |
| **Phụ thuộc** | `Account_Master`, `PageManager`, `ResourcePool`, `AppFactory`, `StatusBar`, `Port/Display.h` (INIT_DONE) |
| **Sử dụng bởi** | `main.cpp` |

### `AppFactory`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Concrete page factory — tạo instance page từ tên class (Template, SystemInfos, Startup). |
| **Files** | `src/App/Pages/AppFactory.h/.cpp` |
| **Public API** | `AppFactory::CreatePage(name)` → `PageBase*` |
| **Phụ thuộc** | `_Template/Template.h`, `SystemInfos/SystemInfos.h`, `StartUp/StartUp.h`, `PageManager/PageFactory.h` |
| **Sử dụng bởi** | `App.cpp` (PageManager khởi tạo với factory này) |

### `Account_Master` / Accounts nodes

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Khởi tạo tất cả Account nodes từ `_ACT_LIST.inc`. Cung cấp AccountBroker trung tâm. |
| **Files** | `src/App/Accounts/Account_Master.h/.cpp`, `ACT_Def.h`, `_ACT_LIST.inc` |
| **Public API** | `Accounts_Init()`, `AccountSystem::Broker()`, `AccountSystem::IMU_Commit(info)` |
| **Phụ thuộc** | `AccountSystem`, `HAL_Def.h`, `ACT_Def.h` |
| **Sử dụng bởi** | `App.cpp`, tất cả ACT_* files |

### `ACT_Storage`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Account node quản lý storage: load/save JSON config từ SD card, quản lý map tile range. |
| **File** | `src/App/Accounts/ACT_Storage.cpp` |
| **Public API** | `onEvent(account, param)` — xử lý STORAGE_CMD_LOAD/SAVE/ADD/REMOVE |
| **Phụ thuộc** | `StorageService`, `MapConv` (set dir path + level), `HAL_SD`, `lv_fs` |
| **Dependents** | `SysConfig` (subscribe nó), `StatusBar` (pull SD info) |

### `ACT_Power`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Account node quản lý power: pull battery info từ HAL, filter (Hysteresis + MedianQueue), timer 500ms kiểm tra charging status. |
| **File** | `src/App/Accounts/ACT_Power.cpp` |
| **Public API** | `onEvent` — xử lý EVENT_TIMER + EVENT_SUB_PULL |
| **Phụ thuộc** | `HAL_Power`, `Filters` (Hysteresis, MedianQueue), `MusicPlayer` (thông báo sạc) |
| **Dependents** | `StatusBar` (pull Power), `SystemInfosModel` (pull Power) |

### `ACT_IMU`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Account node cho IMU: lưu global pointer để `IMU_Commit()` từ HAL có thể push dữ liệu. |
| **File** | `src/App/Accounts/ACT_IMU.cpp` |
| **Public API** | `IMU_Commit(info)` (gọi bởi HAL), `onEvent` (subscriber pull) |
| **Phụ thuộc** | `Account_Master`, `HAL_Def` (IMU_Info_t) |
| **Dependents** | `HAL_IMU` (commit), `SystemInfosModel` (pull) |

### `ACT_MusicPlayer`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Account node play nhạc: nhận notify với tên bài → gọi `HAL::Audio_PlayMusic()`. |
| **File** | `src/App/Accounts/ACT_MusicPlayer.cpp` |
| **Public API** | `onEvent` — xử lý EVENT_NOTIFY với MusicPlayer_Info_t |
| **Phụ thuộc** | `HAL_Audio` |
| **Dependents** | `ACT_Power` (subscribe nó), `StartUpModel` (notify), `HAL_Encoder` (shutdown sound) |

### `ACT_SysConfig`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Account node quản lý cấu hình hệ thống: load/save sound enable, GPS default, language, map dir, WGS84 flag, arrow theme. Subscribe Storage. |
| **File** | `src/App/Accounts/ACT_SysConfig.cpp` |
| **Public API** | `onEvent` — xử lý EVENT_NOTIFY (LOAD) + EVENT_SUB_PULL (trả config) |
| **Phụ thuộc** | `Storage` (subscribe), `HAL_Buzz` (set enable), `Config.h`, macro `STORAGE_VALUE_REG` |
| **Dependents** | `ACT_Storage` (pull SysConfig), App (notify LOAD khi init) |

### `Page::StartUp`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Page khởi động: hiển thị logo "DUMMY" với animation, disable encoder, tự động chuyển sang Template sau 2 giây. |
| **Files** | `src/App/Pages/StartUp/StartUp.h/.cpp`, `StartUpView.h/.cpp`, `StartUpModel.h/.cpp` |
| **Public API** | MVC: Controller (Startup), View (StartUpView), Model (StartUpModel) — lifecycle callbacks |
| **Phụ thuộc** | `PageBase`, `AccountSystem`, `ResourcePool` (font), `lv_ext` (anim timeline), `HAL_Encoder` (disable) |
| **Dependents** | Không (là page khởi đầu) |

### `Page::Template`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Page mẫu: hiển thị thông tin debug (tick count), có canvas vẽ, nhấn để vào SystemInfos. Cache ON. |
| **Files** | `src/App/Pages/_Template/Template.h/.cpp`, `TemplateView.h/.cpp`, `TemplateModel.h/.cpp` |
| **Public API** | MVC: Controller (Template), View (TemplateView), Model (TemplateModel) |
| **Phụ thuộc** | `PageBase`, `ResourcePool` |
| **Dependents** | Không |

### `Page::SystemInfos`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Page hiển thị thông tin hệ thống: Joints, Pose6D, IMU, Battery, Storage, System version. Timer 100ms cập nhật. |
| **Files** | `src/App/Pages/SystemInfos/SystemInfos.h/.cpp`, `SystemInfosView.h/.cpp`, `SystemInfosModel.h/.cpp` |
| **Public API** | MVC lifecycle: pull dữ liệu từ Account system (IMU, Power, Storage), hiển thị lên LVGL |
| **Phụ thuộc** | `PageBase`, `AccountSystem` (pull IMU, Power, Storage), `Version.h`, `StatusBar` (set style), `ResourcePool` |
| **Dependents** | Template page nhấn → Push SystemInfos |

### `Page::StatusBar`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Thanh trạng thái (top-layer overlay): hiển thị SD card detect, Bluetooth, battery %, recorder label. Timer 1 giây pull dữ liệu. |
| **Files** | `src/App/Pages/StatusBar/StatusBar.h/.cpp` |
| **Public API** | `StatusBar::Init(par)`, `StatusBar::Appear(en)`, `StatusBar::SetStyle(style)`, `ACCOUNT_INIT_DEF(StatusBar)` |
| **Phụ thuộc** | `AccountSystem` (pull Power, Storage, nhận notify), `HAL_Bluetooth`, `ResourcePool` (icons), `lvgl.h` |
| **Dependents** | `App.cpp` (Init), `StartUp::onViewDidDisappear` (Appear), `SystemInfos` (SetStyle) |

### `ResourcePool`

| Thuộc tính | Mô tả |
|---|---|
| **Chức năng** | Quản lý tài nguyên: fonts (5) + images (27). Import từ C array. |
| **Files** | `src/App/Resources/ResourcePool.h/.cpp`, `Font/*.c`, `Image/img_src_*.c` |
| **Public API** | `Resource.Init()`, `Resource.GetFont(name)`, `Resource.GetImage(name)` |
| **Phụ thuộc** | `lvgl.h`, `ResourceManager` (PageManager utils) |
| **Sử dụng bởi** | Tất cả Pages (StatusBar, StartUp, Template, SystemInfos) |

---

## 7. Tổng hợp dependency graph

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  main.cpp                                                                  │
│  ├── HAL/                                                                  │
│  │   ├── HAL.cpp (Init/Update coordinator)                                │
│  │   ├── HAL_Backlight.cpp  ← [ledc, lv_anim]                            │
│  │   ├── HAL_IMU.cpp        ← [MPU9250 lib, ACT_IMU]                     │
│  │   ├── HAL_Power.cpp      ← [ADC, Backlight]                           │
│  │   ├── HAL_Encoder.cpp    ← [ButtonEvent, ACT_Encoder*, Buzz, Power]   │
│  │   ├── HAL_Buzz.cpp       ← [ledc, FreeRTOS thread]                    │
│  │   ├── HAL_Audio.cpp      ← [TonePlayer, MusicCode, Buzz]              │
│  │   ├── HAL_SdCard.cpp     ← [SPI, SD lib]                              │
│  │   ├── HAL_Bluetooth.cpp  ← [NimBLE lib]                               │
│  │   └── HAL_I2C_Scan.cpp   ← [Wire]                                     │
│  │                                                                        │
│  ├── Port/                                                                │
│  │   ├── Display.cpp        ← [TFT_eSPI, lvgl, FreeRTOS]                 │
│  │   ├── lv_port_disp.cpp   ← [lvgl, TFT_eSPI]                           │
│  │   ├── lv_port_indev.cpp  ← [lvgl, HAL_Encoder]                        │
│  │   └── lv_port_fatfs.cpp  ← [FatFS, lvgl, SD]                         │
│  │                                                                        │
│  └── App/                                                                 │
│      ├── App.cpp                                                          │
│      │   ├── Accounts/                                                    │
│      │   │   ├── Account_Master.cpp                                       │
│      │   │   │   ├── _ACT_LIST.inc                                        │
│      │   │   │   ├── ACT_Storage.cpp   ← [StorageService, MapConv]       │
│      │   │   │   ├── ACT_Power.cpp     ← [Filters, HAL_Power]            │
│      │   │   │   ├── ACT_IMU.cpp       ← [HAL_Def]                       │
│      │   │   │   ├── ACT_MusicPlayer.cpp ← [HAL_Audio]                   │
│      │   │   │   └── ACT_SysConfig.cpp ← [Storage, HAL_Buzz]             │
│      │   │   └── ACT_Def.h                                                │
│      │   │                                                                │
│      │   ├── Pages/                                                       │
│      │   │   ├── AppFactory.cpp                                           │
│      │   │   ├── StartUp/       → [PageBase, Account, lv_ext]             │
│      │   │   ├── Template/      → [PageBase, ResourcePool]                │
│      │   │   ├── SystemInfos/   → [PageBase, Account (IMU,Power,Storage)] │
│      │   │   └── StatusBar/     → [Account (Power,Storage), ResourcePool] │
│      │   │                                                                │
│      │   ├── Resources/                                                   │
│      │   │   └── ResourcePool.cpp → Fonts (5), Images (27)                │
│      │   │                                                                │
│      │   └── Utils/                                                       │
│      │       ├── AccountSystem/  ← [lvgl timer, vector]                   │
│      │       ├── PageManager/    ← [lvgl, vector, stack]                  │
│      │       ├── Filters/        ← [template headers]                     │
│      │       ├── MapConv/        ← [TileSystem, GPS_Transform]            │
│      │       ├── TileConv/                                                 │
│      │       ├── GPX/ + GPX_Parser/                                       │
│      │       ├── TrackFilter/   ← [math]                                  │
│      │       ├── TonePlayer/    ← [callback]                              │
│      │       ├── ButtonEvent/   ← [stdint]                                │
│      │       ├── StorageService/ ← [ArduinoJson, vector]                  │
│      │       ├── Time/                                                     │
│      │       ├── lv_ext/        ← [lvgl]                                  │
│      │       ├── lv_allocator/  ← [lvgl]                                  │
│      │       └── 3DEngine/      ← [fixed-point math]                      │
│      │                                                                │
│      └── Configs/                                                     │
│          ├── Config.h  ← used by EVERYTHING                           │
│          └── Version.h ← used by HAL.cpp, SystemInfos                 │
│                                                                        │
├── 3rd Party Libraries (lib/ and Libraries/)                             │
│   ├── TFT_eSPI           ← [SPI LCD driver]                            │
│   ├── ArduinoJson        ← [JSON parser]                               │
│   ├── NimBLE             ← [Bluetooth LE stack]                        │
│   ├── FATFS (FF)         ← [File system]                               │
│   └── MPU9250            ← [IMU sensor driver]                         │
│                                                                        │
└── External Tools (ngoài firmware)                                      │
    ├── 3.Software/ImageConvertor/  ← Python tool: ảnh → C array         │
    └── 3.Software/Simulator/       ← Windows LVGL Simulator (VS)       │
```

---

## 8. Bảng dependency 2 chiều (module → depends on → used by)

| Module | Depends On | Used By |
|---|---|---|
| **Config.h** | — | Mọi module |
| **Version.h** | lvgl.h | HAL.cpp, SystemInfos |
| **CommonMacro.h** | — | HAL.cpp, HAL_Power, ACT_Power |
| **HAL (coordinator)** | Tất cả HAL sub-modules, Config.h | main.cpp |
| **HAL_Backlight** | Config, ledc, lv_anim | HAL::Init, Port_Init, Power |
| **HAL_IMU** | MPU9250, I2C, ACCT_IMU | HAL::Init/Update |
| **HAL_Power** | Config, ADC, Backlight | HAL::Init/Update, ACT_Power |
| **HAL_Encoder** | Config, ButtonEvent, Buzz, Power, Account | HAL::Init/Update, lv_port_indev |
| **HAL_Buzz** | Config, ledc, FreeRTOS | HAL::Init, Audio, Encoder, SysConfig |
| **HAL_Audio** | TonePlayer, MusicCode, Buzz | HAL::Init, ACT_MusicPlayer |
| **HAL_SdCard** | Config, SPI, SD lib | HAL::Init/Update, ACT_Storage |
| **HAL_Bluetooth** | NimBLE, Config | HAL::Init/Update, StatusBar |
| **HAL_I2C_Scan** | Wire, Config | HAL::Init |
| **Display / Port** | TFT_eSPI, lvgl, FreeRTOS, Backlight | main.cpp, App.h |
| **lv_port_disp** | lvgl, Config | Port_Init |
| **lv_port_indev** | lvgl, HAL_Encoder | Port_Init |
| **lv_port_fatfs** | FatFS, lvgl, SD | Port_Init |
| **AccountSystem** | lvgl timer, vector | Account_Master, HAL_Encoder |
| **Account_Master** | AccountSystem, ACT_Def | App.cpp |
| **ACT_Storage** | AccountSystem, StorageService, MapConv, HAL_SD | Account_Master init |
| **ACT_Power** | AccountSystem, Filters, HAL_Power, MusicPlayer | Account_Master init |
| **ACT_IMU** | AccountSystem, HAL_Def | Account_Master init, HAL_IMU |
| **ACT_MusicPlayer** | AccountSystem, HAL_Audio | Account_Master init |
| **ACT_SysConfig** | AccountSystem, Storage, HAL_Buzz | Account_Master init |
| **PageManager** | lvgl, vector, stack | App.cpp, tất cả Pages |
| **PageBase** | lvgl | PageManager, Pages |
| **AppFactory** | Page classes, PageFactory | App.cpp |
| **ResourcePool** | lvgl, ResourceManager | App.cpp, Pages |
| **Page::StartUp** | PageBase, Account, lv_ext, Encoder | AppFactory |
| **Page::Template** | PageBase, ResourcePool | AppFactory |
| **Page::SystemInfos** | PageBase, Account, Version, StatusBar | AppFactory |
| **Page::StatusBar** | AccountSystem, HAL_BT, ResourcePool, lvgl | App.cpp, StartUp, SystemInfos |
| **Filters** | (template-only) | ACT_Power |
| **MapConv** | TileSystem, GPS_Transform, lv_fs, math | ACT_Storage |
| **TileConv** | — | (future map module) |
| **GPX / GPX_Parser** | File I/O | (future track module) |
| **TrackFilter** | math | (future track module) |
| **TonePlayer** | callback function | HAL_Audio |
| **ButtonEvent** | stdint, tick | HAL_Encoder |
| **StorageService** | ArduinoJson, vector | ACT_Storage |
| **Time** | — | (future clock/GPS) |
| **lv_ext** | lvgl | StartUpView, pages |
| **lv_allocator** | lvgl | (optional) |
| **3DEngine** | math, stdint | (future Scene3D page) |

---

## 9. Module groups theo layer

```
Layer 1: Hardware Drivers (HAL)
├── HAL_Backlight
├── HAL_IMU
├── HAL_Power
├── HAL_Encoder
├── HAL_Buzz
├── HAL_Audio
├── HAL_SdCard
├── HAL_Bluetooth
└── HAL_I2C_Scan

Layer 2: Platform Port (Port)
├── Display / TFT_eSPI
├── lv_port_disp
├── lv_port_indev
└── lv_port_fatfs

Layer 3: Framework Core
├── AccountSystem (Pub/Sub middleware)
├── PageManager (Page lifecycle)
├── ResourceManager (caching)

Layer 4: Framework Utilities
├── Filters (signal processing)
├── MapConv, TileConv (map/GPS)
├── GPX, GPX_Parser (track data)
├── TrackFilter (path simplification)
├── TonePlayer (melody playback)
├── ButtonEvent (button driver)
├── StorageService (JSON config)
├── Time (clock library)
├── lv_ext (LVGL extensions)
├── lv_allocator (custom allocator)
└── 3DEngine (software 3D renderer)

Layer 5: Application Logic (App)
├── Account_Master (broker setup)
├── ACT_Storage, ACT_Power, ACT_IMU
├── ACT_MusicPlayer, ACT_SysConfig
├── Page::StartUp
├── Page::Template
├── Page::SystemInfos
├── Page::StatusBar
├── ResourcePool (fonts + images)
└── AppFactory (page creation)

Layer 6: Configuration
├── Config.h (pin definitions, system params)
└── Version.h (firmware info)
```
