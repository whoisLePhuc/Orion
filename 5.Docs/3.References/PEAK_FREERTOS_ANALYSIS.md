# 🔷 Peak Project — FreeRTOS Analysis

> Phân tích toàn bộ việc sử dụng FreeRTOS trong firmware Peak: tasks, synchronization,
> ISR, và giao tiếp giữa các task.
>
> Hardware: ESP32-PICO-D4 (dual-core Xtensa LX6, 240MHz)
> FreeRTOS: ESP-IDF FreeRTOS (Amazon SMP variant)

---

## 1. Các Task

### 1.1 Task Overview

Peak sử dụng FreeRTOS qua **Arduino ESP32 core** và **NimBLE stack**. Tổng cộng có **6 task**:

| Task Name | Loại | Stack (words) | Priority | Core | Created by | Vai trò |
|---|---|---|---|---|---|---|
| **loopTask** | User | 8192 (default) | 1 | 1 | Arduino core | Chạy `setup()` + `loop()` |
| **LvglThread** | User | 20000 (~80KB) | MAX-1 (cao nhất) | 1 | `Port/Display.cpp` | LVGL rendering |
| **BuzzerThread** | User | 800 (~3.2KB) | 1 | 1 | `HAL_Buzz.cpp` | Phát tone buzzer |
| **ble** (host) | NimBLE | NIMBLE_STACK_SIZE | MAX-4 | 0 | NimBLE port | BLE host stack |
| **ll** (link layer) | NimBLE | MIN+400 | MAX-1 | 0 | NimBLE port | BLE controller (nếu có) |
| **Idle** | System | configMINIMAL_STACK_SIZE | 0 | — | FreeRTOS | Task rỗi |

> **Lưu ý**: `loopTask` có thể chạy đồng thời với `LvglThread` trên 2 core khác nhau của ESP32.
> Priority: MAX-1 > MAX-4 > 1 > 0.

### 1.2 `loopTask` — Main Arduino Task

- **File**: Arduino core (`cores/esp32/main.cpp`)
- **Priority**: 1 (Arduino default)
- **Stack**: ~8KB mặc định
- **Chức năng**: Chạy `setup()` một lần, sau đó gọi `loop()` vô hạn
- **Core**: Core 1 (Arduino ESP32 thường chạy trên core 1)

```cpp
// src/main.cpp → chạy trong loopTask
void setup() {
    HAL::Init();       // ~1s (có delay 1s trong Power_Init)
    Port_Init();       // ~50ms
    App_Init();        // ~50ms
}

void loop() {
    HAL::Update();     // đọc cảm biến, push data vào AccountSystem
    delay(20);         // ~50Hz
}
```

### 1.3 `LvglThread` — LVGL Rendering Task

- **File**: `src/Port/Display.cpp` (dòng 27-37)
- **Priority**: `configMAX_PRIORITIES - 1` = **cao nhất** (thường là 24 trên ESP32)
- **Stack**: **20000 words** (~80KB) — stack lớn nhất trong toàn bộ firmware
- **Core**: Core 1 (chạy trên cùng core với loopTask mặc định)
- **Khởi tạo**: Task được tạo trong `Port_Init()` nhưng **blocked ngay lập tức** bởi `ulTaskNotifyTake()`

```cpp
// src/Port/Display.cpp
TaskHandle_t handleTaskLvgl;

void TaskLvglUpdate(void* parameter)
{
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  // BLOCKED — chờ signal

    for (;;)
    {
        lv_task_handler();   // LVGL: xử lý timer, animation, redraw
        delay(5);            // ~200Hz render loop
    }
}
```

> ⚠️ **Critical point**: LVGL rendering task KHÔNG chạy ngay sau khi tạo. Nó phải chờ
> `xTaskNotifyGive(handleTaskLvgl)` được gọi từ `App_Init()` → `INIT_DONE()`.
> Điều này đảm bảo toàn bộ hệ thống (HAL, Port, App, Pages, Accounts) được khởi tạo
> xong trước khi LVGL bắt đầu render.

### 1.4 `BuzzerThread` — Buzzer Task

- **File**: `src/HAL/HAL_Buzz.cpp` (dòng 7-22)
- **Priority**: 1 (thấp, ngang với loopTask)
- **Stack**: 800 words (~3.2KB)
- **Core**: Core 1

```cpp
// src/HAL/HAL_Buzz.cpp
static int32_t duration = 0;
static uint32_t freq = 0;

static void BuzzerThread(void* argument)
{
    for (;;)
    {
        if (duration > 0)
        {
            ledcWriteTone(CONFIG_BUZZ_CHANNEL, freq);
            delay(duration);
            ledcWriteTone(CONFIG_BUZZ_CHANNEL, 0);
            duration = 0;
        }
        delay(50);
    }
}
```

> Task này busy-wait bằng `delay()` khi phát tone — không dùng queue hay semaphore.
> Gây lãng phí CPU nhưng buzzer chỉ dùng ngắn (20-50ms mỗi lần).

### 1.5 NimBLE Tasks

- **File**: `lib/NimBLE/src/porting/npl/freertos/src/nimble_port_freertos.c`
- **Khởi tạo**: `nimble_port_freertos_init(host_task_fn)` được gọi trong `NimBLEDevice::init()`
- **Gọi từ**: `HAL::BT_Init()` → `NimBLEDevice::init("Peak-BLE")`

```c
// NimBLE port
void nimble_port_freertos_init(TaskFunction_t host_task_fn)
{
    // Task LL (Link Layer) — nếu controller được bật
    xTaskCreate(nimble_port_ll_task_func, "ll",
        configMINIMAL_STACK_SIZE + 400, NULL,
        configMAX_PRIORITIES - 1, &ll_task_h);

    // Task Host
    xTaskCreatePinnedToCore(host_task_fn, "ble",
        NIMBLE_STACK_SIZE, NULL,
        configMAX_PRIORITIES - 4, &host_task_h, NIMBLE_CORE);
}
```

| Task | Priority | Core | Tham số |
|---|---|---|---|
| `ll` (Link Layer) | MAX-1 (cao nhất) | Core 0 | Stack: MIN+400 words |
| `ble` (Host) | MAX-4 | Core 0 (`NIMBLE_CORE`) | Stack: `NIMBLE_STACK_SIZE` |

> NimBLE stack chạy trên **Core 0** (core PRO), trong khi loopTask và LvglThread chạy trên
> **Core 1** (core APP). Đây là cách phân chia tối ưu cho ESP32 dual-core.

---

## 2. Priority Table

```
Priority  Level
─────────────────────────────────────────────────────
MAX-1     LvglThread       LVGL rendering (24)
          ll (NimBLE)      BLE link layer
│
MAX-4     ble (NimBLE)     BLE host (20)
│
1         loopTask          Arduino main loop
          BuzzerThread      Buzzer tone
│
0         Idle              FreeRTOS idle
```

### Ý nghĩa priority:
- **MAX-1 (cao nhất)**: **LvglThread** + NimBLE LL — các task real-time cần CPU ngay. LVGL cần priority cao để đảm bảo rendering mượt (60 FPS).
- **MAX-4**: NimBLE host — priority thấp hơn LL, đủ nhanh để xử lý BLE events.
- **1**: loopTask + BuzzerThread — background processing.
- **0**: Idle — chạy khi không có task nào khác.

> **Lưu ý**: Trên ESP32, `configMAX_PRIORITIES` thường là **25**. Do đó:
> - LvglThread / ll: priority **24**
> - ble host: priority **20**
> - loopTask / Buzzer: priority **1**

---

## 3. Queue

### Queue tự định nghĩa (Peak Code)
> **Không có**. Code Peak không tạo queue FreeRTOS nào.

### Queue nội bộ (NimBLE)
NimBLE sử dụng queue nội bộ thông qua lớp `ble_npl_eventq`:
```c
// lib/NimBLE/src/porting/npl/freertos/src/npl_os_freertos.c
struct ble_npl_event*
npl_freertos_eventq_get(struct ble_npl_eventq *evq, ble_npl_time_t tmo)
{
    // Dùng FreeRTOS queue bên trong
    // ... xQueueReceive(...)
}

void
npl_freertos_eventq_put(struct ble_npl_eventq *evq, struct ble_npl_event *ev)
{
    // ... xQueueSend(...)
}
```

### LVGL internal queue
LVGL có thể dùng queue nội bộ nếu `LV_USE_OS` được bật, nhưng trong dự án này
`LV_USE_OS` **không được cấu hình** → LVGL chạy bare-metal trong LvglThread.

---

## 4. Semaphore

### Semaphore tự định nghĩa (Peak Code)
> **Không có**. Code Peak không tạo semaphore FreeRTOS nào.

### Semaphore nội bộ (NimBLE)
NimBLE sử dụng semaphore qua abstraction `npl_freertos_sem_*`:
```c
// lib/NimBLE/src/porting/npl/freertos/src/npl_os_freertos.c
ble_npl_error_t npl_freertos_sem_init(struct ble_npl_sem *sem, uint16_t tokens) {
    sem->handle = xSemaphoreCreateCounting(255, tokens);
    // ...
}
ble_npl_error_t npl_freertos_sem_pend(struct ble_npl_sem *sem, ble_npl_time_t timeout) {
    return xSemaphoreTake(sem->handle, timeout / portTICK_PERIOD_MS);
}
ble_npl_error_t npl_freertos_sem_release(struct ble_npl_sem *sem) {
    xSemaphoreGive(sem->handle);
}
```

---

## 5. Mutex

### Mutex tự định nghĩa (Peak Code)
> **Không có**. Code Peak không dùng mutex FreeRTOS nào.

### Mutex nội bộ (NimBLE)
NimBLE sử dụng mutex qua abstraction `npl_freertos_mutex_*`:
```c
// lib/NimBLE/src/porting/npl/freertos/src/npl_os_freertos.c
ble_npl_error_t npl_freertos_mutex_init(struct ble_npl_mutex *mu) {
    mu->handle = xSemaphoreCreateMutex();
}
ble_npl_error_t npl_freertos_mutex_pend(struct ble_npl_mutex *mu, ble_npl_time_t timeout) {
    return xSemaphoreTake(mu->handle, timeout / portTICK_PERIOD_MS);
}
```

### LVGL mutex
LVGL có thể dùng mutex nếu `LV_USE_OS` được bật, nhưng trong dự án này **không dùng**.

### Shared resource access không có mutex
Một số shared resources được truy cập từ nhiều task **mà không có mutex**:
- **`EncoderDiff`** (volatile int16_t) — ghi từ ISR, đọc từ loopTask qua `lv_port_indev`
- **`duration`** / **`freq`** (static variables) — ghi từ loopTask (`Buzz_Tone`), đọc từ BuzzerThread
- **LVGL internal state** — lý do LvglThread phải là task DUY NHẤT gọi `lv_task_handler()`

---

## 6. Event Group

> **Không sử dụng**. Không có `xEventGroupCreate`, `xEventGroupSetBits`, `xEventGroupWaitBits` nào trong code Peak.

---

## 7. Task Notifications (Cơ chế đồng bộ CHÍNH)

Đây là cơ chế đồng bộ **duy nhất** do Peak code sử dụng trực tiếp:

### Luồng: `INIT_DONE` → Wake LvglThread

```
App_Init() [loopTask]                       LvglThread
─────────────────────────                    ─────────────
                                              ulTaskNotifyTake() → BLOCKED
                                                    │
                                                    │ (chờ vô hạn)
                                                    │
manager.Push("Pages/Startup")
    ...
INIT_DONE() ─────────────────────────────────────► WAKE UP!
                                                      │
xTaskNotifyGive(handleTaskLvgl)                    for (;;) {
                                                       lv_task_handler();
                                                       delay(5);
                                                    }
```

```cpp
// App.h — Init done signal
#define INIT_DONE() \
do { \
    xTaskNotifyGive(handleTaskLvgl); \
} while(0)

// Display.cpp — LVGL task chờ signal
void TaskLvglUpdate(void* parameter) {
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  // BLOCK
    for (;;) {
        lv_task_handler();
        delay(5);
    }
}
```

> **Task Notification** được dùng thay vì semaphore vì nhanh hơn (direct task-to-task,
> không cần kernel object) và chiếm ít RAM hơn.

---

## 8. ISR (Interrupt Service Routines)

### 8.1 Encoder GPIO Interrupt

- **Pin**: `CONFIG_ENCODER_A_PIN` (GPIO 35)
- **Trigger**: `CHANGE` (cạnh lên và xuống)
- **File**: `src/HAL/HAL_Encoder.cpp`

```cpp
// src/HAL/HAL_Encoder.cpp
static volatile int16_t EncoderDiff = 0;

static void Encoder_IrqHandler()
{
    // Đọc trạng thái A, B
    uint8_t a = digitalRead(CONFIG_ENCODER_A_PIN);
    uint8_t b = digitalRead(CONFIG_ENCODER_B_PIN);
    // ...
    // Tính toán chiều quay
    EncoderDiff += (count - countLast) > 0 ? 1 : -1;
}
```

**Đặc điểm ISR encoder:**
- Gọi trực tiếp `digitalRead()` bên trong ISR — chấp nhận được trên ESP32 (có cache)
- **Không có** `portENTER_CRITICAL` / `portEXIT_CRITICAL` — dùng `volatile` để đảm bảo visibility
- Biến `EncoderDiff` được ISR ghi và loopTask đọc → **shared resource không được bảo vệ**
- `attachInterrupt` (Arduino API) wrap xung quanh GPIO ISR của ESP32

### 8.2 Luồng dữ liệu ISR → Task

```
GPIO Interrupt (Encoder A pin)
    │
    ▼
Encoder_IrqHandler()        ← ISR context (Core bất kỳ)
    │
    │ EncoderDiff += delta
    │ volatile int16_t
    ▼
loopTask (HAL::Update)
    │
    ├── Encoder_Update()     ← push button debounce
    │
    ▼
lv_port_indev::encoder_read() ← được gọi bởi LvglThread
    │
    ├── HAL::Encoder_GetDiff() → đọc EncoderDiff, reset về 0
    ├── gọi Encoder_RotateHandler(diff)
    │       → Account("Encoder").Commit(&diff)
    │       → Account("Encoder").Publish()
    │
    └── HAL::Encoder_GetIsPush() → digitalRead(PUSH_PIN)
```

> ⚠️ **Vấn đề**: `EncoderDiff` được ghi từ ISR (core bất kỳ) và đọc từ `loopTask` (core 1)
> không có mutex/critical section. Vì kiểu `int16_t` là atomic trên Xtensa LX6, điều này
> an toàn trong thực tế nhưng không đúng strict-SMP.

---

## 9. Giao tiếp giữa các task

### 9.1 Task Interaction Diagram

```
  Core 0 (PRO)                  Core 1 (APP)
────────────────────────      ────────────────────────
                              ┌──────────────────┐
                              │   loopTask        │
                              │   Priority: 1     │
                              │                    │
  ┌──────────────────┐        │  HAL::Update()     │
  │  ble (NimBLE)    │        │  ├─ IMU_Update()  │
  │  Priority: MAX-4 │        │  ├─ Encoder_Update│
  │                   │        │  ├─ Power_Update  │
  │  ┌─────────────┐  │        │  ├─ Audio_Update  │
  │  │ NimBLE que/  │  │        │  ├─ BT_Update()─►│───► ble task
  │  │ sem/mutex   │  │        │  └─ SD_Update()  │
  │  └─────────────┘  │        └────────┬─────────┘
  └──────────────────┘                 │
                                       │ Account::Commit() / Publish()
                                       ▼
                              ┌──────────────────┐
  ┌──────────────────┐        │  LvglThread       │
  │  ll (NimBLE)    │        │  Priority: MAX-1  │
  │  Priority: MAX-1│        │                    │
  └──────────────────┘        │  lv_task_handler()│
                              │  ├─ Timer callbacks│
  ┌──────────────────┐        │  ├─ Animation     │
  │  BuzzerThread    │        │  ├─ Input handling│
  │  Priority: 1     │        │  │  └─ encoder_rea│
  │                   │        │  ├─ Redraw dirty  │
  │  Buzz_Tone()◄────│────────│─── from loopTask  │
  │  [shared vars]   │        │  └─ disp_flush_cb │
  └──────────────────┘        └──────────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │  TFT_eSPI (SPI)   │
                              │  ST7789 LCD       │
                              └──────────────────┘
```

### 9.2 Cơ chế giao tiếp

| Cơ chế | Từ → Đến | Dữ liệu |
|---|---|---|
| **Task Notification** | loopTask → LvglThread | Signal (1 lần, boot) |
| **Shared volatile** | ISR → loopTask | `EncoderDiff` (int16) |
| **AccountSystem** (Pub/Sub) | loopTask → (trong task) | IMU, Power data |
| **Shared variables** | loopTask → BuzzerThread | `freq`, `duration` |
| **NimBLE eventq** | NimBLE task → NimBLE task | BLE events |

### 9.3 Giao tiếp nội bộ trong cùng task (loopTask)

Phần lớn giao tiếp trong Peak xảy ra **trong cùng loopTask** thông qua **AccountSystem**:

```
loopTask:
  HAL::Update()
    ├── HAL::IMU_Update()
    │     → AccountSystem::IMU_Commit(&IMU_Info_t)    [Commit data]
    │     → Account("IMU") doit commit vào PingPongBuffer
    │
    └── HAL::Encoder_Update() → chạy trong loopTask
          → Account("Encoder").Commit(&diff)  [Commit data]
          → Account("Encoder").Publish()      [Push đến subscribers]

  LvglThread gọi lv_task_handler():
    → LVGL timer StatusBar_Update (1000ms):
        actStatusBar->Pull("Storage", &sdInfo)
        actStatusBar->Pull("Power", &powerInfo)
```

> **Quan trọng**: AccountSystem **không có mutex** — việc Commit/Publish/Pull chỉ an toàn
> vì tất cả đều chạy trong cùng loopTask. StatusBar timer (LvglThread) Pull dữ liệu trong
> khi loopTask Commit dữ liệu → **đây là race condition tiềm ẩn**.
>
> Tuy nhiên, vì HAL::Update và lv_task_handler không chạy đồng thời (loopTask delay 20ms
> tạo cơ hội cho LvglThread chạy giữa các lần HAL::Update), dữ liệu thường nhất quán.

---

## 10. Timeline đồng bộ task

```
ESP32 Boot
    │
    ├── FreeRTOS kernel init
    │
    ├── Arduino core tạo loopTask (priority 1)
    │
    ├── loopTask chạy setup()
    │
    ├── HAL::Init()
    │   └── Buzz_init() → xTaskCreate(BuzzerThread)     ← priority 1
    │   └── BT_Init()    → NimBLE tạo ll + ble tasks    ← priority MAX-1, MAX-4
    │
    ├── Port_Init()
    │   └── xTaskCreate(LvglThread)                     ← priority MAX-1
    │       └── Task BLOCKED ngay ở ulTaskNotifyTake()
    │
    ├── App_Init()
    │   ├── manager.Push("Pages/Startup")
    │   │   → Page lifecycle bắt đầu (state LOAD)
    │   │   → LVGL objects được tạo (root, label, animation)
    │   │   → State WILL_APPEAR → object visible
    │   │   ⚠️ LvglThread chưa chạy → màn hình chưa được render!
    │   └── INIT_DONE()
    │       → xTaskNotifyGive(handleTaskLvgl)
    │       → LvglThread WAKE UP
    │       → LvglThread bắt đầu: lv_task_handler() loop
    │       → LVGL render startup page lên màn hình 🎬
    │
    ├── loop() bắt đầu
    │   └── HAL::Update() + delay(20) (50Hz)
    │
    └── LvglThread loop:
        └── lv_task_handler() + delay(5) (200Hz)
```

---

## 11. Tồn tại / Lưu ý

### ⚠️ Race Condition tiềm ẩn

1. **EncoderDiff** — ISR ghi, loopTask đọc không có critical section:
   ```c
   // ISR context (core bất kỳ)
   EncoderDiff += 1;
   
   // Task context (core 1)
   int16_t diff = -EncoderDiff / 2;  // có thể đọc giá trị không nhất quán
   EncoderDiff = 0;                   // race nếu ISR xảy ra giữa 2 câu lệnh
   ```
   
   **Mức độ ảnh hưởng**: Thấp (mất 1-2 xung encoder, người dùng không nhận thấy)

2. **Buzz shared variables** — loopTask ghi `freq`/`duration`, BuzzerThread đọc:
   ```c
   // loopTask: HAL::Buzz_Tone(500, 20)
   freq = 500;
   duration = 20;
   
   // BuzzerThread
   if (duration > 0) {
       ledcWriteTone(CONFIG_BUZZ_CHANNEL, freq);  // race: freq chưa kịp set
   }
   ```
   
   **Mức độ ảnh hưởng**: Thấp (tone sai hiếm khi xảy ra)

3. **AccountSystem cross-task access** — loopTask Commit, LvglThread Pull:
   ```
   loopTask:  Account("Power").Push → gọi subscriber callback
   LvglThread: StatusBar timer → Account("Power").Pull
   ```
   
   **Mức độ ảnh hưởng**: Thấp (xảy ra ở các thời điểm khác nhau)

### Tóm tắt sử dụng FreeRTOS primitives

| Primitive | Peak code | NimBLE | LVGL |
|---|---|---|---|
| `xTaskCreate` | ✅ 2 tasks | ✅ 2 tasks | ❌ |
| Task Notification | ✅ 1 usage | ❌ | ❌ |
| Queue | ❌ | ✅ (eventq) | ❌ (bare metal) |
| Semaphore | ❌ | ✅ (counting) | ❌ |
| Mutex | ❌ | ✅ (mutex) | ❌ |
| Event Group | ❌ | ❌ | ❌ |
| Critical Section | ❌ | ✅ (npl_freertos) | ❌ |
| Software Timer | ❌ | ❌ | ✅ (LVGL timer) |
| `delay()` / `vTaskDelay` | ✅ | ❌ | ❌ |

---

## 12. Sơ đồ task interaction tổng thể

```
    ╔══════════════════════════════════════════════════════════╗
    ║                   Core 0 (PRO)                          ║
    ║                                                          ║
    ║  ┌──────────────────────────────────────────────┐       ║
    ║  │  NimBLE Host Task ("ble")                    │       ║
    ║  │  Prio: MAX-4, Stack: NIMBLE_STACK_SIZE       │       ║
    ║  │                                              │       ║
    ║  │  nimble_port_run()                           │       ║
    ║  │  ├─ xQueueReceive(evq)  ← đợi BLE events    │       ║
    ║  │  ├─ xSemaphoreTake(...) ← mutex cho host     │       ║
    ║  │  └─ ble_gap_conn_cb                          │       ║
    ║  └──────────────────────────────────────────────┘       ║
    ║                                                          ║
    ║  ┌──────────────────────────────────────────────┐       ║
    ║  │  NimBLE LL Task ("ll")                        │       ║
    ║  │  Prio: MAX-1                                 │       ║
    ║  │  └─ BLE controller (HCI)                     │       ║
    ║  └──────────────────────────────────────────────┘       ║
    ╚══════════════════════════════════════════════════════════╝

    ╔══════════════════════════════════════════════════════════╗
    ║                   Core 1 (APP)                          ║
    ║                                                          ║
    ║  ┌──────────────────────────────────────────────┐       ║
    ║  │  loopTask (Arduino)                           │       ║
    ║  │  Prio: 1                                     │       ║
    ║  │                                              │       ║
    ║  │  setup() → [HAL::Init, Port_Init, App_Init]  │       ║
    ║  │  loop() → [HAL::Update, delay(20)]           │       ║
    ║  │                                              │       ║
    ║  │  HAL::Update():                              │       ║
    ║  │  ├── IMU: I2C_read → Commit("IMU")           │       ║
    ║  │  ├── Encoder: ISR diff → Commit("Encoder")   │       ║
    ║  │  ├── Power: ADC_read → Pull("Power")         │       ║
    ║  │  └── BT: BT_Update → connect                  │       ║
    ║  └──────────────────────────────────────────────┘       ║
    ║                                                          ║
    ║  ┌──────────────────────────────────────────────┐       ║
    ║  │  LvglThread                                   │       ║
    ║  │  Prio: MAX-1 (cao nhất), Stack: 80KB         │       ║
    ║  │                                              │       ║
    ║  │  for (;;) {                                  │       ║
    ║  │    lv_task_handler();   ← LVGL engine        │       ║
    ║  │    delay(5);           ← yield               │       ║
    ║  │  }                                           │       ║
    ║  │                                              │       ║
    ║  │  lv_task_handler() xử lý:                    │       ║
    ║  │  ├── LVGL timers: StatusBar, Page timers     │       ║
    ║  │  ├── Animation: Push/Pop trang, fade         │       ║
    ║  │  ├── Input: encoder_read() → HAL_Encoder     │       ║
    ║  │  └── Display flush: disp_flush_cb → SPI      │       ║
    ║  └──────────────────────────────────────────────┘       ║
    ║                                                          ║
    ║  ┌──────────────────────────────────────────────┐       ║
    ║  │  BuzzerThread                                 │       ║
    ║  │  Prio: 1, Stack: 800 words                   │       ║
    ║  │                                              │       ║
    ║  │  for (;;) {                                  │       ║
    ║  │    if (duration > 0) {                       │       ║
    ║  │      ledcWriteTone(freq);                    │       ║
    ║  │      delay(duration);                        │       ║
    ║  │      ledcWriteTone(0);                       │       ║
    ║  │    }                                         │       ║
    ║  │    delay(50);                                │       ║
    ║  │  }                                           │       ║
    ║  └──────────────────────────────────────────────┘       ║
    ║                                                          ║
    ║  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐     ║
    ║     ISR: GPIO Interrupt (Encoder A pin)                  ║
    ║     └─ Encoder_IrqHandler() → volatile EncoderDiff      ║
    ║  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘     ║
    ╚══════════════════════════════════════════════════════════╝
```

---

## 13. Kết luận

1. **Peak sử dụng FreeRTOS tối thiểu**: Chỉ có 2 user tasks (`LvglThread`, `BuzzerThread`)
   + 2 NimBLE tasks (`ble`, `ll`).

2. **Cơ chế đồng bộ chính**: `xTaskNotifyGive`/`ulTaskNotifyTake` để đánh thức LvglThread
   sau khi boot xong. Phần còn lại dùng `delay()` + polling.

3. **Không có Queue / Semaphore / Mutex / Event Group nào do Peak code tạo** — toàn bộ
   giao tiếp trong cùng loopTask qua AccountSystem (Pub/Sub).

4. **ISR duy nhất**: Encoder GPIO interrupt, giao tiếp với task qua biến `volatile`.

5. **Race condition tiềm ẩn**: 3 vị trí (Encoder, Buzz, AccountSystem cross-task) nhưng
   mức độ ảnh hưởng thấp.

6. **Phân bố core**: Core 0 chạy NimBLE BLE stack, Core 1 chạy application + LVGL
   — đây là cách phân chia hợp lý cho ESP32 dual-core.
