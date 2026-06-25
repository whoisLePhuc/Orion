# 🔷 Peak Project — Message Framework Analysis (AccountSystem)

> Phân tích chi tiết Message Framework (AccountSystem) — xương sống giao tiếp
> nội bộ của Peak firmware. Thiết kế theo mô hình Pub/Sub với cơ chế
> Ping-Pong Buffer và Event Dispatch đồng bộ.

---

## 1. Message là gì?

**Message** trong Peak là một gói dữ liệu có cấu trúc được truyền giữa các
module (Account nodes) thông qua `AccountSystem`. Mỗi message bao gồm:

```cpp
// Account.h — EventParam_t
typedef struct {
    EventCode_t event;   // Loại sự kiện (PUBLISH, PULL, NOTIFY, TIMER)
    Account* tran;       // Account gửi (sender)
    Account* recv;       // Account nhận (receiver)
    void* data_p;        // Con trỏ tới dữ liệu
    uint32_t size;       // Kích thước dữ liệu (bytes)
} EventParam_t;
```

**Đặc điểm cốt lõi:**

| Thuộc tính | Giá trị |
|---|---|
| **Mô hình** | Pub/Sub đồng bộ (synchronous) |
| **Cơ chế** | Gọi callback trực tiếp — không qua queue |
| **Thread** | **Không thread-safe** — tất cả trong cùng loopTask |
| **Hàng đợi** | Không có message queue |
| **Buffer** | Ping-Pong (double buffer) cho dữ liệu được cache |
| **Kết nối** | Account-to-Account trực tiếp (qua vector pointers) |

---

## 2. Publisher

### Định nghĩa

**Publisher** (publisher) là Account chủ động tạo dữ liệu và thông báo
cho các subscriber. Một account trở thành publisher khi các account khác
`Subscribe()` vào nó.

Để làm publisher, một account cần:
1. Có **buffer size > 0** khi khởi tạo (để chứa dữ liệu cache)
2. Gọi `Commit(data, size)` để ghi dữ liệu
3. Gọi `Publish()` để đẩy dữ liệu đến tất cả subscribers

### Các Publisher trong Peak

| Publisher | Buffer | Dữ liệu | Commit bởi |
|---|---|---|---|
| **IMU** | `sizeof(IMU_Info_t)` (36 bytes × 2) | ax, ay, az, gx, gy, gz, mx, my, mz, roll, pitch, yaw | `HAL::IMU_Update()` |
| **Encoder** | `sizeof(int16_t)` (2 bytes × 2) | encoder diff value | `HAL::Encoder_GetDiff()` |
| **Power** | 0 (không cache — pull-through-callback) | voltage, usage, isCharging | Pull callback |
| **Storage** | 0 (không cache) | SD detect, size, config data | Pull/Notify callback |
| **SysConfig** | 0 (không cache) | sound, GPS default, language, map... | Pull callback |

> **Lưu ý**: Account có buffer (Encoded, IMU) là **publisher chủ động**.
> Account không buffer (Power, Storage, SysConfig) vẫn là **publisher bị động**
> — chỉ trả dữ liệu khi subscriber `Pull()` qua callback.

### Cơ chế hoạt động của Publisher

```
// Bước 1: Tạo Account với buffer
// ACT_IMU.cpp
Account("IMU", &dataCenter, sizeof(HAL::IMU_Info_t));

// Bước 2: Commit dữ liệu (ghi vào PingPongBuffer)
// HAL_IMU.cpp
void AccountSystem::IMU_Commit(HAL::IMU_Info_t* info) {
    actIMU->Commit(info, sizeof(HAL::IMU_Info_t));   // Ghi buffer
}

// Bước 3: Publish đến subscribers
// HAL_Encoder.cpp
actEncoder->Commit(&diff, sizeof(int16_t));
actEncoder->Publish();     // Đẩy đến tất cả subscribers
```

---

## 3. Subscriber

### Định nghĩa

**Subscriber** (subscriber) là Account đăng ký nhận dữ liệu từ một publisher.
Một account subscribe vào publisher khác qua `Subscribe(pubID)`.

```cpp
Account* Subscribe(const char* pubID);
```

Hậu quả của `Subscribe()`:
1. Publisher được thêm vào vector `publishers` của subscriber
2. Subscriber được thêm vào vector `subscribers` của publisher
3. Subscriber có thể `Pull()` dữ liệu hoặc nhận `Publish()` push

### Các Subscriber trong Peak

| Subscriber | Subscribe vào | Lý do |
|---|---|---|
| **Power** | `MusicPlayer` | Thông báo khi bắt đầu/kết thúc sạc → play sound |
| **SysConfig** | `Storage` | Lưu/tải cấu hình qua Storage |
| **Storage** | `SysConfig` | Pull SysConfig để cấu hình MapConv |
| **StatusBar** | `Power`, `Storage` | Pull battery + SD status mỗi giây |
| **SystemInfosModel** | `IMU`, `Power`, `Storage` | Pull sensor data hiển thị UI |
| **StartUpModel** | — (không subscribe) | Chỉ Notify MusicPlayer |
| **AccountMaster** | **Tất cả** (auto) | Dùng cho broadcast notify |

### Cơ chế Subscribe

```cpp
// ACT_Power.cpp — Power subscribe vào MusicPlayer
ACCOUNT_INIT_DEF(Power) {
    account->Subscribe("MusicPlayer");
    account->SetEventCallback(onEvent);
    account->SetTimerPeriod(500);
}

// ACT_Storage.cpp — Storage subscribe vào SysConfig
ACCOUNT_INIT_DEF(Storage) {
    account->SetEventCallback(onEvent);
    account->Subscribe("SysConfig");
}
```

**Mô hình kết nối subscriber-publisher (đồ thị có hướng):**

```
    IMU ──────► SystemInfosModel
     │
     ├────────► AccountMaster (auto)

    Encoder ──► (dành cho PageManager navigation)

    Power ────► StatusBar
     │
     ├────────► SystemInfosModel
     │
     ├────────► AccountMaster (auto)
     │
     └────────► MusicPlayer (subscribe ngược)

    Storage ──► StatusBar
     │
     ├────────► SystemInfosModel
     │
     ├────────► AccountMaster (auto)
     │
     └────────► SysConfig (subscribe ngược)

    MusicPlayer
     │
     ├────────► Power (subscribe ngược)
     │
     └────────► AccountMaster (auto)

    SysConfig ─► Storage (subscribe ngược)
     │
     └──────────► AccountMaster (auto)
```

> **Mũi tên**: Subscriber → Publisher (Subscribe)
> **Chiều dữ liệu**: Publisher → Subscriber (Publish / Pull)

---

## 4. Event Dispatch

### 4.1 Event Types

```cpp
typedef enum {
    EVENT_NONE,
    EVENT_PUB_PUBLISH,   // Publisher đẩy dữ liệu → subscriber callback
    EVENT_SUB_PULL,      // Subscriber kéo dữ liệu → publisher callback
    EVENT_NOTIFY,        // Subscriber gửi lệnh → publisher callback
    EVENT_TIMER,         // Timer định kỳ → chính account đó
    _EVENT_LAST
} EventCode_t;
```

### 4.2 Cơ chế Dispatch

**AccountSystem dispatch đồng bộ** — event callback được gọi trực tiếp,
không qua queue, không qua ISR, không context switch:

```
Publisher::Publish()
    │
    │  param.event = EVENT_PUB_PUBLISH
    │  param.tran  = Publisher
    │  param.data_p = PingPongBuffer read pointer
    │  param.size  = BufferSize
    │
    ├── for each subscriber in subscribers[]
    │       │
    │       ├── callback = subscriber->priv.eventCallback
    │       │
    │       ├── if callback != nullptr:
    │       │       param.recv = subscriber
    │       │       int ret = callback(subscriber, &param)  ← gọi TRỰC TIẾP
    │       │       └── subscriber xử lý dữ liệu
    │       │
    │       └── if callback == nullptr:
    │               bỏ qua (subscriber không quan tâm)
    │
    └── PingPongBuffer_SetReadDone()
```

```
Subscriber::Pull(pubID)
    │
    │  param.event = EVENT_SUB_PULL
    │  param.tran  = subscriber
    │  param.recv  = publisher
    │  param.data_p = buffer (subscriber cấp)
    │  param.size  = kích thước yêu cầu
    │
    ├── callback = publisher->priv.eventCallback
    │
    ├── if callback != nullptr:
    │       int ret = callback(publisher, &param)
    │       └── publisher ghi dữ liệu vào param.data_p
    │
    └── if callback == nullptr:
            Đọc trực tiếp từ PingPongBuffer cache
```

```
Subscriber::Notify(pubID, data)
    │
    │  param.event = EVENT_NOTIFY
    │  param.tran  = subscriber
    │  param.recv  = publisher
    │  param.data_p = data (subscriber cấp)
    │  param.size  = size
    │
    └── callback = publisher->priv.eventCallback
        int ret = callback(publisher, &param)
        └── publisher xử lý lệnh/command
```

```
TimerCallbackHandler (LVGL timer)
    │
    │  param.event = EVENT_TIMER
    │  param.tran  = account (chính nó)
    │  param.recv  = account (chính nó)
    │  param.data_p = nullptr
    │
    └── callback(account, &param)
```

### 4.3 Error Handling

Hệ thống trả về các mã lỗi:

```cpp
typedef enum {
    ERROR_NONE                =  0,   // Thành công
    ERROR_UNKNOW              = -1,   // Lỗi không xác định
    ERROR_SIZE_MISMATCH       = -2,   // Kích thước dữ liệu không khớp
    ERROR_UNSUPPORTED_REQUEST = -3,   // Event type không hỗ trợ
    ERROR_NO_CALLBACK         = -4,   // Không có event callback
    ERROR_NO_CACHE            = -5,   // Không có buffer cache
    ERROR_NO_COMMITED         = -6,   // Chưa commit dữ liệu
    ERROR_NOT_FOUND           = -7,   // Không tìm thấy publisher
    ERROR_PARAM_ERROR         = -8    // Tham số sai
} ErrorCode_t;
```

---

## 5. Message Queue

> **Không có message queue trong AccountSystem.**

### Giải thích

AccountSystem không sử dụng hàng đợi (queue) vì:

1. **Tất cả operations đồng bộ trong cùng task**:
   - `HAL::Update()` chạy trong `loopTask`
   - `lv_task_handler()` → StatusBar timer Pull cũng trong `LvglThread`
   - Không có cross-thread messaging

2. **Không cần buffer cho messages**:
   - `EVENT_PUB_PUBLISH`: chỉ gọi callback trực tiếp
   - `EVENT_SUB_PULL`: publisher trả dữ liệu ngay lập tức
   - `EVENT_NOTIFY`: gọi callback publisher ngay lập tức

3. **PingPongBuffer chỉ dùng cho cache, không phải queue**:
   - 2 buffer ping-pong (double buffer)
   - Writer ghi vào buffer A, reader đọc từ buffer B
   - Khi writer done → flag lên, reader có thể đọc
   - Kích thước cố định (cấp phát tại constructor)

### So sánh với Message Queue truyền thống

| Đặc điểm | AccountSystem | Message Queue truyền thống |
|---|---|---|
| **Hàng đợi** | ❌ Không | ✅ FIFO queue |
| **Xử lý** | Đồng bộ (callback ngay) | Bất đồng bộ (gửi → xử lý sau) |
| **Blocking** | Block caller | Non-blocking (gửi rồi đi tiếp) |
| **Bộ nhớ** | Tối thiểu (chỉ callback) | Queue buffer |
| **Thread-safety** | ❌ Không | ✅ Có thể |

---

## 6. Message Flow

### 6.1 Flow 1: Publish (Push — dữ liệu cảm biến)

**Ví dụ thực tế**: IMU đẩy dữ liệu đến SystemInfos page

```
loopTask (50Hz)
    │
    ├── HAL::IMU_Update()
    │     │
    │     ├── MPU9250.update()    ← I2C đọc cảm biến
    │     ├── Điền IMU_Info_t {ax, ay, az, gx, gy, gz, mx, my, mz, roll, pitch, yaw}
    │     │
    │     └── AccountSystem::IMU_Commit(&imuInfo)
    │           │
    │           └── Account("IMU")::Commit(data, sizeof(IMU_Info_t))
    │                 │
    │                 ├── PingPongBuffer_GetWriteBuf() → lấy buffer ghi
    │                 ├── memcpy(wBuf, data, size)     → ghi dữ liệu
    │                 └── PingPongBuffer_SetWriteDone() → flag có dữ liệu mới
    │
    └── (HAL::Update kết thúc — IMU chưa Publish, chỉ Commit)

LvglThread (200Hz)
    │
    └── lv_task_handler()
          │
          ├── SystemInfos timer (100ms) fire
          │     │
          │     ├── SystemInfos::Update()
          │     │     │
          │     │     ├── Model.GetIMUInfo(buf) → SystemInfosModel
          │     │     │     │
          │     │     │     └── account->Pull("IMU", &imu, sizeof(imu))
          │     │     │           │
          │     │     │           └── Account::Pull(Account* pub)
          │     │     │                 │
          │     │     │                 ├── pub = Account("IMU")
          │     │     │                 ├── callback = pub->priv.eventCallback
          │     │     │                 │
          │     │     │                 ├── vì ACT_IMU::onEvent = nullptr
          │     │     │                 │   (IMU account không có eventCallback)
          │     │     │                 │
          │     │     │                 └── Đọc trực tiếp PingPongBuffer:
          │     │     │                       PingPongBuffer_GetReadBuf()
          │     │     │                       memcpy(data_p, rBuf, size)
          │     │     │                       PingPongBuffer_SetReadDone()
          │     │     │
          │     │     └── View.SetIMU(buf)    ← update LVGL label
          │     │
          │     └── StatusBar timer (1000ms) fire
          │           │
          │           └── actStatusBar->Pull("Power", &power, sizeof(power))
          │                 │
          │                 └── Account("Power")::Pull()
          │                       │
          │                       ├── callback = ACT_Power::onEvent
          │                       │
          │                       └── callback(pub, &param):
          │                             │
          │                             ├── param.event = EVENT_SUB_PULL
          │                             ├── HAL::Power_GetInfo(&powerInfo)
          │                             ├── Filter::Hysteresis + MedianQueue
          │                             └── memcpy(param.data_p, &powerInfo, size)
```

### 6.2 Flow 2: Notify (Command — gửi lệnh điều khiển)

**Ví dụ thực tế 1**: App_Init gửi lệnh LOAD đến Storage

```cpp
// App_Init()
ACCOUNT_SEND_NOTIFY_CMD(Storage, STORAGE_CMD_LOAD);
// Mở rộng thành:
// AccountSystem::Broker()->AccountMaster.Notify("Storage", &info, sizeof(info));
```

```
AccountMaster::Notify("Storage", &info)
    │
    ├── Tìm "Storage" trong publishers của AccountMaster
    │   (AccountMaster tự động subscribe tất cả accounts)
    │
    ├── gọi Account("Storage")::Notify(pub, data, size)
    │     │
    │     ├── param.event = EVENT_NOTIFY
    │     ├── param.tran = AccountMaster
    │     ├── param.recv = Storage
    │     ├── param.data_p = {STORAGE_CMD_LOAD}
    │     │
    │     └── callback = ACT_Storage::onEvent
    │           int ret = callback(Storage, &param)
    │           │
    │           └── onEvent(Storage, &param):
    │                 │
    │                 ├── param.event == EVENT_NOTIFY ✓
    │                 ├── cmd = STORAGE_CMD_LOAD
    │                 │
    │                 ├── storageService.LoadFile()    ← đọc JSON từ SD
    │                 │
    │                 └── account->Pull("SysConfig", &sysConfig, sizeof(sysConfig))
    │                       │
    │                       └── Account("SysConfig")::Pull()
    │                             ├── callback = ACT_SysConfig::onEvent
    │                             ├── EVENT_SUB_PULL
    │                             └── memcpy(data, &sysConfig, size)
    │                                   └── MapConv::SetDirPath(sysConfig.mapDirPath)
    │                                   └── MapConv::SetCoordTransformEnable(!sysConfig.WGS84)
```

**Ví dụ thực tế 2**: Power phát hiện sạc → báo MusicPlayer

```
ACT_Power timer (500ms) fire
    │
    ├── HAL::Power_GetInfo(&power)
    ├── power.isCharging != lastStatus
    │
    └── account->Notify("MusicPlayer", &info, sizeof(info))
          │
          ├── ACT_MusicPlayer::onEvent
          ├── param.event = EVENT_NOTIFY
          ├── param.data_p = {music = "BattChargeStart"}
          │
          └── HAL::Audio_PlayMusic("BattChargeStart")
                └── player.Play(melodyCode)
```

**Ví dụ thực tế 3**: Config load → SysConfig nhận

```cpp
// App_Init()
ACCOUNT_SEND_NOTIFY_CMD(SysConfig, SYSCONFIG_CMD_LOAD);
```

```
AccountMaster → Notify("SysConfig", {SYSCONFIG_CMD_LOAD})
    │
    └── ACT_SysConfig::onEvent
          │
          ├── EVENT_NOTIFY + SYSCONFIG_CMD_LOAD
          │
          └── HAL::Buzz_SetEnable(sysConfig.soundEnable)
                └── Bật/tắt buzzer theo config
```

### 6.3 Flow 3: Timer (Event định kỳ)

**Ví dụ thực tế**: Power timer 500ms kiểm tra sạc

```cpp
// ACT_Power.cpp
ACCOUNT_INIT_DEF(Power) {
    account->SetEventCallback(onEvent);
    account->SetTimerPeriod(500);   // 500ms timer
}
```

```
LVGL timer fire (500ms)
    │
    └── Account::TimerCallbackHandler(timer)
          │
          ├── param.event = EVENT_TIMER
          ├── param.tran = Account("Power")
          ├── param.recv = Account("Power")
          │
          └── ACT_Power::onEvent(Power, &param)
                │
                ├── param.event == EVENT_TIMER ✓
                │
                ├── HAL::Power_GetInfo(&power)
                ├── if (power.isCharging != lastStatus)
                │     account->Notify("MusicPlayer", &info)
                │     // → play "BattChargeStart" / "BattChargeEnd"
                │
                └── lastStatus = power.isCharging
```

### 6.4 Flow 4: Pull (Kéo dữ liệu)

**Ví dụ thực tế**: StatusBar Pull dữ liệu mỗi giây

```
LvglThread — StatusBar_Update timer (1000ms)
    │
    ├── actStatusBar->Pull("Storage", &sdInfo, sizeof(sdInfo))
    │     │
    │     └── Account("Storage")::Pull()
    │           │
    │           ├── callback = ACT_Storage::onEvent
    │           ├── EVENT_SUB_PULL
    │           │
    │           └── onEvent(Storage, &param):
    │                 info->isDetect    = HAL::SD_GetReady()
    │                 info->totalSizeMB = HAL::SD_GetCardSizeMB()
    │                 info->freeSizeMB  = 0.0f
    │
    ├── HAL::BluetoothConnected()
    │     → hiện/ẩn icon BT
    │
    ├── actStatusBar->Pull("Power", &power, sizeof(power))
    │     │
    │     └── ACT_Power::onEvent:
    │           ├── HAL::Power_GetInfo(&powerInfo)
    │           ├── Filter::Hysteresis + MedianQueue
    │           └── memcpy(param.data_p, &powerInfo, size)
    │
    └── Update LVGL:
          ├── ui.imgSD (SD icon)
          ├── ui.imgBT (BT icon)
          ├── ui.battery.label ("85%")
          └── ui.battery.objUsage (bar height)
```

### 6.5 Flow 5: AccountMaster — Broadcast

**AccountMaster** tự động subscribe vào **tất cả** accounts khi chúng được
tạo (trong `AccountBroker::AddAccount`). Dùng để broadcast command:

```cpp
// App.h — broadcast macro
#define ACCOUNT_SEND_NOTIFY_CMD(ACT, CMD) \
do { \
    AccountSystem::ACT##_Info_t info; \
    memset(&info, 0, sizeof(info)); \
    info.cmd = AccountSystem::CMD; \
    AccountSystem::Broker()->AccountMaster.Notify(#ACT, &info, sizeof(info)); \
} while(0)
```

```
Accounts: [Storage, Power, IMU, StatusBar, MusicPlayer, SysConfig]
              ▲       ▲      ▲      ▲            ▲            ▲
              │       │      │      │            │            │
              └───────┴──────┴──────┴────────────┴────────────┘
                                │
                    AccountMaster (auto-subscribe)
                                │
                    AccountSystem::Broker()->AccountMaster
```

---

## 7. PingPongBuffer — Cơ chế Double Buffer

AccountSystem dùng **PingPongBuffer** (double buffer) để lưu cache dữ liệu:

```
Struct: PingPongBuffer_t
    buffer[2] = {buf0, buf1}     // 2 buffers thực tế
    writeIndex                     // Buffer đang ghi (0 hoặc 1)
    readIndex                      // Buffer đang đọc
    readAvaliable[2]               // Flag: buffer đã sẵn sàng đọc chưa
```

```
Luồng Commit + Publish (Push model):

  Writer (HAL::IMU_Update)
    │
    ├── GetWriteBuf()
    │     if writeIndex == readIndex → writeIndex = !readIndex
    │     return buffer[writeIndex]
    │
    ├── memcpy(buffer[writeIndex], data, size)  ← ghi dữ liệu
    │
    ├── SetWriteDone()
    │     readAvaliable[writeIndex] = true
    │     writeIndex = !writeIndex
    │
    └── Publish()
          for subscriber in subscribers[]
              GetReadBuf()
                  if readAvaliable[0]: readIndex=0, return buffer[0]
                  if readAvaliable[1]: readIndex=1, return buffer[1]
                  else: no data
              callback(subscriber, &param)  ← push data
              SetReadDone()
                  readAvaliable[readIndex] = false

  Reader (SystemInfosModel Pull — khi không có callback)
    │
    ├── GetReadBuf()
    ├── memcpy(user_buf, buffer[readIndex], size)
    └── SetReadDone()
```

```
Ví dụ tuần tự:

Thời điểm 0:  buffer[0] rỗng, buffer[1] rỗng
              writeIndex=0, readAvaliable=[false, false]

Commit #1:    writeIndex=0 → ghi data vào buffer[0]
              readAvaliable=[true, false], writeIndex=1

Publish:      readAvaliable[0]=true → đọc buffer[0]
              setReadDone → readAvaliable=[false, false]

Commit #2:    writeIndex=1 (≠ readIndex=0) → ghi vào buffer[1]
              readAvaliable=[false, true], writeIndex=0

Commit #3:    writeIndex=0 (= readIndex=0) → chuyển: writeIndex=1
              → ghi vào buffer[1], ghi đè lên buffer[1] chưa đọc!
              → mất dữ liệu commit #2 (đây là hạn chế của ping-pong)
```

> **Hạn chế**: Nếu Commit nhanh hơn đọc, buffer chưa đọc sẽ bị ghi đè.
> Trong Peak, Commit chỉ có 50Hz (IMU) còn Pull là 100ms (SystemInfos)
> nên không xảy ra mất dữ liệu.

---

## 8. Sơ đồ tổng thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AccountSystem                                │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  AccountBroker "MASTER"                                       │  │
│  │  ├── AccountMaster (tự động subscribe mọi account)            │  │
│  │  └── AccountPool:                                             │  │
│  │        ├── "Storage"     [buf=0]    ─── Sub: SysConfig        │  │
│  │        ├── "Power"       [buf=0]    ─── Sub: MusicPlayer      │  │
│  │        ├── "IMU"         [buf=36]   ─── Sub: (none)           │  │
│  │        ├── "StatusBar"   [buf=0]    ─── Sub: Power, Storage   │  │
│  │        ├── "MusicPlayer" [buf=0]    ─── Sub: (none)           │  │
│  │        └── "SysConfig"   [buf=0]    ─── Sub: Storage          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Account class                                                │  │
│  │                                                                 │
│  │  foreach Account:                                               │  │
│  │  ┌──────────────────────┐                                       │  │
│  │  │  ID: "IMU"          │                                       │  │
│  │  │  UserData: nullptr   │                                       │  │
│  │  │  eventCallback: func │← xử lý EVENT_*                       │  │
│  │  │  timer: lv_timer_t*  │← EVENT_TIMER định kỳ                 │  │
│  │  │  BufferSize: 36      │← kích thước cache                    │  │
│  │  │  BufferManager:      │                                       │  │
│  │  │    ┌ PingPongBuffer ─┤                                       │  │
│  │  │    │ [buf0, buf1]    │                                       │  │
│  │  │    │ writeIndex      │                                       │  │
│  │  │    │ readIndex       │                                       │  │
│  │  │    │ readAvaliable[] │                                       │  │
│  │  │    └─────────────────┘                                       │  │
│  │  │                                                              │  │
│  │  │  publishers[]:                                                │  │
│  │  │    [Account* "MusicPlayer", Account* "StatusBar"...]          │  │
│  │  │                                                              │  │
│  │  │  subscribers[]:                                               │  │
│  │  │    [Account* "SystemInfosModel", Account* "AccountMaster"]    │  │
│  │  └──────────────────────┘                                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Event Dispatch Matrix

| Gọi | Trên Account | event | Ai nhận | Dùng khi |
|---|---|---|---|---|
| `Commit()` + `Publish()` | Publisher | `EVENT_PUB_PUBLISH` | Subscribers | Đẩy dữ liệu cảm biến (IMU, Encoder) |
| `Pull(pubID)` | Subscriber | `EVENT_SUB_PULL` | Publisher's callback | Kéo dữ liệu theo yêu cầu (StatusBar, SystemInfos) |
| `Notify(pubID, data)` | Subscriber | `EVENT_NOTIFY` | Publisher's callback | Gửi lệnh / command (Storage LOAD, MusicPlayer play) |
| Timer fire | Chính nó | `EVENT_TIMER` | Chính nó | Kiểm tra định kỳ (Power check sạc) |

---

## 10. Ví dụ thực tế — Luồng xử lý Encoder

```
Physical Encoder xoay
    │
    ├── GPIO Interrupt
    │     └── Encoder_IrqHandler() → EncoderDiff++
    │
    ├── lv_port_indev::encoder_read() [LvglThread]
    │     └── HAL::Encoder_GetDiff()
    │           ├── diff = -EncoderDiff / 2
    │           ├── EncoderDiff = 0
    │           ├── Encoder_RotateHandler(diff)
    │           │     ├── HAL::Buzz_Tone(300, 5)
    │           │     └── Account("Encoder").Commit(&diff, 2)
    │           │           └── PingPongBuffer: ghi diff vào buffer
    │           └── return diff
    │
    ├── LV_INDEV_TYPE_ENCODER → group focus change
    │     └── SystemInfosView::onFocus()
    │           └── scroll to focused item
    │
    └── LVGL redraw: focus indicator (style transition)
          └── disp_flush_cb() → SPI → ST7789
```

---

## 11. Tổng kết

| Đặc điểm | Giá trị |
|---|---|
| **Kiểu** | Pub/Sub đồng bộ, synchronous callback |
| **Thread-safety** | ❌ Không (single-threaded assumption) |
| **Hàng đợi** | ❌ Không |
| **Buffer** | PingPong (double buffer) cho cached publishers |
| **Kết nối** | Account-to-Account (vector pointers) |
| **Broadcast** | AccountMaster auto-subscribe tất cả |
| **Timers** | LVGL timer → `EVENT_TIMER` |
| **Dữ liệu** | Struct nhị phân (`IMU_Info_t`, `Power_Info_t`, ...) |
| **Số Account** | 7 (6 data + 1 Master) |
| **Số Event types** | 4 (PUBLISH, PULL, NOTIFY, TIMER) |
| **Files** | `Account.h/cpp`, `AccountBroker.h/cpp`, `PingPongBuffer.h/c` |
