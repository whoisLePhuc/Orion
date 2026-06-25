# 🔷 Peak Project — GUI Architecture Analysis

> Phân tích kiến trúc giao diện đồ họa (GUI) sử dụng LVGL v8.1 trên nền tảng ESP32-PICO-D4.

---

## 1. Tổng quan

Peak sử dụng **LVGL v8.1** (Light and Versatile Graphics Library) làm thư viện GUI chính.
Kiến trúc GUI được tổ chức theo mô hình **MVC** (Model-View-Controller) với PageManager
quản lý vòng đời các màn hình.

```
┌─────────────────────────────────────────────────────────────┐
│                    GUI Architecture                          │
│                                                             │
│  LVGL Core (lv_task_handler)                                │
│  ├── Screen: lv_scr_act()     ← active screen               │
│  ├── Layer: lv_layer_top()    ← overlay (StatusBar)         │
│  └── Layers → Object Tree (lv_obj parent-child)             │
│                                                             │
│  PageManager (Lifecycle Stack)                              │
│  ├── PageBase ← Startup, Template, SystemInfos...           │
│  │   ├── Controller (Page*.cpp)    ← lifecycle + events     │
│  │   ├── View (Page*View.cpp)      ← LVGL widgets           │
│  │   └── Model (Page*Model.cpp)    ← AccountSystem data     │
│  └── StatusBar (top layer overlay)                          │
│                                                             │
│  AccountSystem (Pub/Sub Data) ← Model kéo dữ liệu            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Screen

LVGL có khái niệm **Screen** (màn hình) — là root container chứa tất cả objects.
Peak sử dụng **1 active screen duy nhất** trong suốt vòng đời.

### Khởi tạo Screen

```cpp
// ResourcePool.cpp — khi App_Init()
lv_obj_remove_style_all(lv_scr_act());    // Xóa style mặc định
lv_disp_set_bg_color(lv_disp_get_default(), lv_color_black());  // Nền đen
```

**Đặc điểm:**
- Chỉ có **1 screen** (`lv_scr_act()`) — không tạo screen mới
- Tất cả page được tạo như **con của screen** (`lv_obj_create(lv_scr_act())`)
- Screen background: **màu đen** (LV_COLOR_BLACK)
- PageManager tạo root object cho mỗi page:
  ```cpp
  // PM_State.cpp — StateLoadExecute()
  lv_obj_t* root_obj = lv_obj_create(lv_scr_act());
  lv_obj_set_size(root_obj, LV_HOR_RES, LV_VER_RES);   // 240×240
  lv_obj_clear_flag(root_obj, LV_OBJ_FLAG_SCROLLABLE);
  base->root = root_obj;   // ← page lưu root
  ```

### Layer

Có **2 layer objects** trên màn hình:

| Layer | Hàm LVGL | Mục đích | Parent của |
|---|---|---|---|
| **Active Screen** | `lv_scr_act()` | Nội dung chính | Root objects của các Page |
| **Top Layer** | `lv_layer_top()` | Overlay | StatusBar container |

```
lv_scr_act() ─────────────────────────────────────────────────┐
│  Page Stack (z-order từ dưới lên):                          │
│                                                             │
│  [Page Base] root object (không scrollable, 240×240)        │
│  ├── Page content widgets (labels, images, containers...)   │
│                                                             │
│  [Page Top] root object (nếu có, được move_foreground())    │
│  └── ...                                                    │
└─────────────────────────────────────────────────────────────┘

lv_layer_top() ───────────────────────────────────────────────┐
│  StatusBar container (240×22, y=-22, trượt xuống khi show)  │
│  ├── satellite icon                                         │
│  ├── SD card icon                                           │
│  ├── Bluetooth icon                                         │
│  ├── recorder label                                         │
│  └── battery icon + label + usage bar                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Page

**Page** là đơn vị UI chính — tương đương một "màn hình" trong ứng dụng.
Mỗi page kế thừa `PageBase` và có vòng đời riêng được quản lý bởi **PageManager**
dạng stack (giống iOS UINavigationController).

### PageBase — Lớp cơ sở

```cpp
// Utils/PageManager/PageBase.h
class PageBase {
public:
    lv_obj_t* root;           // LVGL root object (con của screen)
    PageManager* Manager;     // Page manager
    const char* Name;         // Tên page ("Startup", "Template",...)
    uint16_t ID;
    void* UserData;

    // Lifecycle callbacks — 8 states
    virtual void onCustomAttrConfig();   // Cấu hình cache/animation
    virtual void onViewLoad();           // Khởi tạo view
    virtual void onViewDidLoad();        // Load xong
    virtual void onViewWillAppear();     // Trước khi hiện
    virtual void onViewDidAppear();      // Đã hiện
    virtual void onViewWillDisappear();  // Chuẩn bị ẩn
    virtual void onViewDidDisappear();   // Đã ẩn
    virtual void onViewDidUnload();      // Hủy
};
```

### Các Page đã cài đặt

| Page | File | Chức năng | Cache | Animation |
|---|---|---|---|---|
| **StartUp** | `Pages/StartUp/` | Logo + animation khởi động, 2s → Template | OFF | NONE |
| **Template** | `Pages/_Template/` | Page mẫu debug (tick, arm image), nhấn → SystemInfos | ON | OVER_BOTTOM, bounce |
| **SystemInfos** | `Pages/SystemInfos/` | 6 item scroll: Joints, Pose6D, System, IMU, Battery, Storage | Default | Fade in |
| **StatusBar** | `Pages/StatusBar/` | Thanh trạng thái overlay (top layer) | N/A | Slide từ trên xuống |

> `SystemInfos` có reference đến `Push("Pages/Scene3D")` — dành cho 3D engine trong tương lai.

---

## 4. Widget

Các LVGL widget được sử dụng:

| Widget | Sử dụng trong | Mục đích |
|---|---|---|
| `lv_obj` | Tất cả | Container, root, item wrapper |
| `lv_label` | Tất cả | Text: title, info, data values |
| `lv_img` | StatusBar, Template, SystemInfos | Icons: battery, BT, SD, satellite, arm, joints... |
| `lv_btn` | StatusBar (gián tiếp) | Có thể dùng cho sau này |
| `lv_canvas` | Template | Vẽ đồ họa tùy chỉnh |
| `lv_group` | SystemInfos, Template | Nhóm các object để encoder focus |

### Widget tree mẫu — SystemInfos Page

```
root (240×240, flex column, bg=black)
├── item_t "Joints"  (cont 220px)
│   ├── icon (flex column, có style.focus khi focused)
│   │   ├── img "joints"
│   │   └── label "Joints"
│   ├── labelInfo "J1\nJ2\nJ3\nJ4\nJ5\nJ6"
│   └── labelData  (giá trị số)
├── item_t "Pose6D"  (cont 220px)
│   ├── icon (flex column)
│   │   ├── img "pose6d"
│   │   └── label "Pose6D"
│   ├── labelInfo "X\nY\nZ\nA\nB\nC"
│   └── labelData
├── item_t "System"   (cont 220px)
│   └── ...
├── item_t "IMU"      (cont 220px)
│   └── ...
├── item_t "Battery"  (cont 220px)
│   └── ...
└── item_t "Storage"  (cont 220px)
    └── ...
```

### Widget tree — StatusBar

```
cont (240×22, y=0 hoặc -22, LV_ALIGN_TOP_MID)
├── img "satellite"
├── label "0" (số vệ tinh)
├── img "sd_card" (hidden khi không có SD)
├── img "bluetooth" (hidden khi không kết nối)
├── label "REC" (hidden khi không record)
├── img "battery"
│   └── obj (usage bar, màu trắng, chiều cao tỉ lệ %)
└── label "100%" (phần trăm pin)
```

### Widget tree — Template Page

```
root (240×240)
├── label "title" (LV_ALIGN_TOP_MID, font 14)
├── label "tick = xxx" (LV_ALIGN_TOP_MID, font 10)
└── img "arm" (center, group focus)
```

### Widget tree — StartUp Page

```
root (240×240)
└── cont (110×40, center, border-bottom đỏ 3px)
    └── label "DUMMY" (font montserrat 26, white, center)
        ← animation: width 0→110, label y từ dưới lên
```

---

## 5. Event

### 5.1 Event types

Peak sử dụng **4 loại event**:

| Loại event | Cơ chế | Mô tả |
|---|---|---|
| **LVGL Event** | `lv_event_cb` | Tương tác người dùng (nhấn, xoay encoder) |
| **PageManager Event** | Callback lifecycle | State transitions của Page |
| **AccountSystem Event** | Pub/Sub message | Dữ liệu giữa các module |
| **LVGL Timer** | `lv_timer_cb` | Cập nhật UI định kỳ |

### 5.2 LVGL Events (User Input)

Input device duy nhất: **Encoder** (xoay + nhấn).

```
Encoder Event Flow:

Physical Encoder
    │
    ├── Xoay
    │     → GPIO Interrupt → EncoderDiff (volatile)
    │     → lv_port_indev::encoder_read()
    │       → HAL::Encoder_GetDiff() → trả về số bước
    │       → dữ liệu đưa vào LV_INDEV_TYPE_ENCODER
    │
    └── Nhấn
          → HAL::Encoder_GetIsPush()
          → LV_INDEV_STATE_PR / LV_INDEV_STATE_REL
          → LV_EVENT_PRESSED / LV_EVENT_RELEASED trên object focused
```

Binding event với widget:

```cpp
// SystemInfos::onViewLoad() — gắn event cho các icon item
AttachEvent(root);
AttachEvent(View.ui.joints.icon);
AttachEvent(View.ui.pose6d.icon);
// ...

void SystemInfos::AttachEvent(lv_obj_t* obj) {
    lv_obj_set_user_data(obj, this);
    lv_obj_add_event_cb(obj, onEvent, LV_EVENT_PRESSED, this);
}

void SystemInfos::onEvent(lv_event_t* event) {
    lv_obj_t* obj = lv_event_get_target(event);
    lv_event_code_t code = lv_event_get_code(event);
    if (code == LV_EVENT_PRESSED) {
        // Nhấn vào pose6d.icon → chuyển sang Scene3D
        instance->Manager->Push("Pages/Scene3D");
    }
}
```

### 5.3 LVGL Group & Focus (Encoder Navigation)

Encoder xoay để chuyển focus giữa các items trong group:

```
Encoder Xoay
    → lv_port_indev (LV_INDEV_TYPE_ENCODER)
    → LV_GROUP focused object thay đổi
    → onFocus callback gọi scroll để item vào tầm nhìn
    → Style transition: width 70→220 với overshoot animation
```

```cpp
// SystemInfosView — Group setup
ui.group = lv_group_create();
lv_group_set_focus_cb(ui.group, onFocus);
lv_indev_set_group(lv_get_indev(LV_INDEV_TYPE_ENCODER), ui.group);
lv_group_add_obj(ui.group, ui.joints.icon);
lv_group_add_obj(ui.group, ui.pose6d.icon);
// ... 6 items

void SystemInfosView::onFocus(lv_group_t* g) {
    lv_obj_t* icon = lv_group_get_focused(g);
    lv_obj_t* cont = lv_obj_get_parent(icon);
    lv_coord_t y = lv_obj_get_y(cont);
    lv_obj_scroll_to_y(lv_obj_get_parent(cont), y, LV_ANIM_ON);
}
```

### 5.4 PageManager Events (Lifecycle)

```
PageManager::Push("Pages/StartUp")
    │
    ├── SwitchAnimStateCheck() → OK
    ├── FindPageInStack() → chưa có
    ├── FindPageInPool("Startup") → tìm thấy
    ├── PageStack.push(base)
    │
    └── SwitchTo(base, isPush=true)
          │
          ├── PageCurrent = startup
          ├── PageCurrent->priv.State = LOAD (vì chưa cache)
          │
          ├── StateUpdate(PagePrev=null) → bỏ qua
          │
          └── StateUpdate(PageCurrent):
                ┌─────────────────────────────────────┐
                │  LOAD                               │
                │  StateLoadExecute():                │
                │  ├─ root = lv_obj_create(screen)    │
                │  ├─ onViewLoad() ← tạo UI            │
                │  └─ onViewDidLoad()                  │
                │  return WILL_APPEAR                  │
                ├─────────────────────────────────────┤
                │  WILL_APPEAR                        │
                │  onViewWillAppear() ← start anim    │
                │  SwitchAnimCreate()                 │
                │  return DID_APPEAR                  │
                ├─────────────────────────────────────┤
                │  "Đang chờ animation hoàn tất..."    │
                │  (LVGL anim callback → onAnimFinish) │
                │  → StateUpdate(base)                 │
                ├─────────────────────────────────────┤
                │  DID_APPEAR                         │
                │  onViewDidAppear()                   │
                │  return ACTIVITY                     │
                ├─────────────────────────────────────┤
                │  ACTIVITY  ← page đang hiển thị    │
                │  (chờ timer hoặc event để Push/Pop)  │
                └─────────────────────────────────────┘
```

### 5.5 AccountSystem Events (Data)

Không phải LVGL event, nhưng là nguồn dữ liệu cho UI update:

```
HAL::IMU_Update()
  → Account("IMU").Commit(&imuInfo)    // Ghi vào PingPongBuffer
  → Account("IMU").Publish()           // Gửi EVENT_PUB_PUBLISH

SystemInfos timer (100ms)
  → Model.GetIMUInfo()
    → account->Pull("IMU", &imu, sizeof(imu))
      → Account("IMU") gửi EVENT_SUB_PULL
      → callback trả dữ liệu
  → View.SetIMU(info)                  // Cập nhật label
```

### 5.6 LVGL Timers

| Timer | Period | File | Chức năng |
|---|---|---|---|
| StartUp onTimer | 2000ms (1 lần) | `StartUp.cpp` | Chuyển sang Template page |
| StatusBar Update | 1000ms | `StatusBar.cpp` | Pull + update SD, BT, battery |
| SystemInfos Update | 100ms | `SystemInfos.cpp` | Pull + update IMU, Power, Storage |
| Template onTimer | 100ms | `Template.cpp` | Update tick count |

---

## 6. Navigation

### 6.1 Cơ chế Stack (Push/Pop)

PageManager sử dụng **std::stack** để quản lý navigation:

```
Khởi tạo: Stack rỗng
    │
    ├── Push("Pages/StartUp")
    │   Stack: [StartUp]
    │
    ├── [2s timer] → Push("Pages/Template")
    │   Stack: [StartUp, Template]  → StartUp bị đẩy xuống
    │   → StartUp lifecycle: WILL_DISAPPEAR → DID_DISAPPEAR → UNLOAD
    │   Stack: [Template]  ← StartUp bị pop khỏi stack
    │
    ├── Nhấn nút → Push("Pages/SystemInfos")
    │   Stack: [Template, SystemInfos]
    │   → Template lifecycle: WILL_DISAPPEAR → DID_DISAPPEAR (cached)
    │   → SystemInfos lifecycle: LOAD → WILL_APPEAR → DID_APPEAR
    │
    └── Pop() ← quay lại
        Stack: [Template]  ← SystemInfos pop khỏi stack
        → SystemInfos lifecycle: WILL_DISAPPEAR → DID_DISAPPEAR → UNLOAD
        → Template lifecycle: WILL_APPEAR → DID_APPEAR (từ cache)
```

### 6.2 Animation Types

| Animation | Hướng | Thời gian | Đường cong |
|---|---|---|---|
| **Global (mặc định)** | OVER_TOP | 500ms | ease_out |
| **Template** | OVER_BOTTOM | 500ms | bounce |
| **StartUp** | NONE | 0 | — |
| **SystemInfos** | Global OVER_TOP | 500ms | ease_out |

Animation system:
```
Push → Page mới OVER từ trên xuống, đè lên page cũ
Pop  → Page cũ OVER xuống dưới, page mới từ cache hiện ra

Animation được LVGL anim quản lý, callback onSwitchAnimFinish
khi hoàn tất → StateUpdate() chuyển sang DID_APPEAR / DID_DISAPPEAR
```

### 6.3 Root Drag (Swipe)

PageManager hỗ trợ drag (swipe) để chuyển page — code trong `PM_Drag.cpp`:

```cpp
// Gắn LV_EVENT_PRESSED, LV_EVENT_PRESSING, LV_EVENT_RELEASED vào root
// Nếu drag vượt ngưỡng → Pop() page hiện tại
```

Tuy nhiên, drag chưa được kích hoạt trong code hiện tại (không page nào Enable).

---

## 7. MVC Pattern

Mỗi page tuân theo mô hình **MVC** (Model-View-Controller) với 3 files:

```
Page::StartUp
├── StartUp.h/.cpp       ← Controller: lifecycle + logic
├── StartUpView.h/.cpp   ← View: LVGL widgets
└── StartUpModel.h/.cpp  ← Model: Account + data access
```

### Mô hình MVC

```
  ┌─────────────────────────────────────────────────┐
  │                    Controller                    │
  │                (Page*.cpp)                       │
  │                                                  │
  │  Lifecycle: onViewLoad → onViewWillAppear → ...  │
  │  Logic: timer, event handling                    │
  │  Liên kết: View.Create(root)                     │
  │            Model.Init()                           │
  │            Model.Pull() → View.Set*(data)        │
  └────┬──────────────┬──────────────────────────────┘
       │              │
       ▼              ▼
  ┌──────────┐   ┌──────────────────────────────────┐
  │   View   │   │            Model                  │
  │          │   │                                   │
  │  LVGL    │   │  Account*(AccountSystem)          │
  │  objects │   │  Pull/Publish data                │
  │  styles  │   │  Subcribe("IMU")                  │
  │  layout  │   │  GetIMUInfo()                     │
  └──────────┘   └──────────────────────────────────┘
```

### Controller (Page*.cpp)

Điều phối toàn bộ: khởi tạo View + Model, xử lý lifecycle events, timer, và
LVGL events. Ví dụ `SystemInfos`:

```cpp
class SystemInfos : public PageBase {
    // Các lifecycle override
    void onViewLoad() override {
        Model.Init();                // Khởi tạo Model → Subscribe Accounts
        View.Create(root);           // Khởi tạo View → tạo LVGL widgets
        AttachEvent(...);            // Gắn event cho các icon
    }
    void onViewWillAppear() override {
        lv_indev_set_group(...);     // Chuyển group focus
        timer = lv_timer_create(...); // 100ms update timer
    }
    void Update() {                  // Gọi từ timer
        Model.GetIMUInfo(buf);
        View.SetIMU(buf);            // Model → View
        Model.GetBatteryInfo(...);
        View.SetBattery(...);
    }
    static void onEvent(lv_event_t* e) {
        if (LV_EVENT_PRESSED)
            Manager->Push("Pages/Scene3D");  // Navigation
    }
};
```

### View (Page*View.h/.cpp)

Chỉ chứa LVGL objects và style. Public API là các `Set*()` method:

```cpp
class SystemInfosView {
    void Create(lv_obj_t* root);     // Tạo toàn bộ widget tree
    void Delete();                    // Xóa styles
    // Public Setters
    void SetIMU(const char* info);
    void SetBattery(int usage, float voltage, const char* state);
    void SetStorage(const char* detect, const char* size, const char* ver);
private:
    struct { lv_style_t icon, focus, info, data; } style;
    void Item_Create(item_t* item, lv_obj_t* par, const char* name, ...);
    void Group_Init();
};
```

### Model (Page*Model.h/.cpp)

Chứa dữ liệu và logic truy xuất qua AccountSystem:

```cpp
class SystemInfosModel {
    Account* account;
    void Init() {
        account = new Account("SystemInfosModel", Broker(), 0, this);
        account->Subscribe("IMU");       // Subscribe data nodes
        account->Subscribe("Power");
        account->Subscribe("Storage");
    }
    void GetIMUInfo(char* info, uint32_t len) {
        HAL::IMU_Info_t imu;
        account->Pull("IMU", &imu, sizeof(imu));  // Pull từ AccountSystem
        snprintf(info, len, "%.3f\n%.3f\n...");
    }
};
```

### MVC ở mức Page — tổng quan

```
Mỗi Page là 1 MVC instance độc lập.

PageManager quản lý nhiều Page,
mỗi Page có MVC riêng.

┌──────────────────────────────────────────────────────┐
│  PageManager (Stack)                                 │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Page::Template          (đang active)        │   │
│  │  ├─ Controller: lifecycle + event             │   │
│  │  ├─ View:  labelTitle, labelTick, img canvas  │   │
│  │  └─ Model: GetData() → lv_tick_get()          │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Page::SystemInfos    (đã cache)              │   │
│  │  ├─ Controller: 100ms timer update           │   │
│  │  ├─ View: 6 item + group focus              │   │
│  │  └─ Model: Pull IMU, Power, Storage          │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  StatusBar (Top Layer, độc lập với stack)    │   │
│  │  ├─ Controller: 1000ms timer                 │   │
│  │  ├─ View: battery, SD, BT icons              │   │
│  │  └─ Model: Pull Power, Storage (inline)      │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

> **Lưu ý**: StatusBar không phải PageBase kế thừa MVC. Nó là namespace với
> các hàm static, đăng ký Account riêng (`ACT_StatusBar`), nhưng logic
> update nằm trong static callback timer.

---

## 8. Luồng tạo màn hình

### Luồng đầy đủ từ boot → widget hiển thị

```
ESP32 Boot
    │
    ├── lv_init()                           ← LVGL core init
    │
    ├── lv_port_disp_init()                 ← Đăng ký display driver
    │   └── flush_cb → TFT_eSPI::pushColors
    │   └── draw buffer: 240×120 × 2 bytes (half screen)
    │
    ├── lv_port_indev_init()                ← Đăng ký input driver
    │   └── LV_INDEV_TYPE_ENCODER
    │
    ├── ResourcePool::Init()                ← Cấu hình screen
    │   ├── lv_obj_remove_style_all(lv_scr_act())
    │   ├── lv_disp_set_bg_color(black)
    │   └── Import fonts + images vào ResourceManager
    │
    ├── StatusBar::Init(lv_layer_top())     ← Tạo top layer UI
    │   └── lv_timer_create(StatusBar_Update, 1000ms)
    │
    ├── PageManager::Install pages × 3      ← Đăng ký page vào pool
    │   └── Factory::CreatePage(name)
    │   └── base->onCustomAttrConfig()      ← Cấu hình page
    │
    ├── PageManager::Push("Pages/StartUp")  ← Bắt đầu page lifecycle
    │   └── StateLoadExecute():
    │       root = lv_obj_create(lv_scr_act())
    │       root size = 240×240, no scroll
    │       Startup::onViewLoad():
    │           StartUpView::Create(root)
    │           ├── lv_obj_create → cont (110×40, center)
    │           ├── lv_label_create → "DUMMY"
    │           └── lv_anim_timeline_create (width + y animation)
    │           lv_timer_create(onTimer, 2000)
    │
    │   └── StateWillAppearExecute():
    │       Startup::onViewWillAppear()
    │       → lv_anim_timeline_start (width: 0→110, label y slide)
    │       → root visible!
    │
    ├── INIT_DONE() → LvglThread WAKE UP    ← LVGL bắt đầu render
    │
    │   === MÀN HÌNH STARTUP HIỂN THỊ ===
    │
    ├── LvglThread loop:
    │   for (;;) {
    │       lv_task_handler();
    │       ├── Animation timeline chạy → cont width nở ra
    │       ├── Label "DUMMY" trượt lên
    │       ├── Flush display → SPI → ST7789
    │       │
    │       ├── [2s sau] Startup timer fire
    │       │   → Manager->Push("Pages/Template")
    │       │   → StartUp fade out (onViewDidLoad: fade_out 1500ms)
    │       │   → StartUp WILL_DISAPPEAR → DID_DISAPPEAR
    │       │     → StatusBar::Appear(true) ← trượt xuống
    │       │   → StartUp UNLOAD → root deleted
    │       │
    │       ├── Template WILL_APPEAR
    │       │   → Animation OVER_BOTTOM bounce
    │       │   → labelTitle, labelTick, img arm hiển thị
    │       │
    │       └── [100ms timer] Template update → labelTick
    │   }
    │
    └── loopTask:
        HAL::Update() → cập nhật cảm biến
        delay(20)
```

---

## 9. Cây giao diện tổng thể

```
Display (ST7789, 240×240, 16-bit, 60FPS)
│
└── LVGL Display Driver
    │
    └── lv_scr_act() ─────────────────────────────────────────
        │                                                     │
        │  Background: black (lv_disp_set_bg_color)           │
        │                                                     │
        ├── [Page: Startup root]  (khi active, 240×240)      │
        │   └── cont (110×40, centered, border-bottom đỏ)    │
        │       └── label "DUMMY" (font 26, white)           │
        │                                                     │
        ├── [Page: Template root]  (khi active, 240×240)     │
        │   ├── label "Title" (TOP_MID, y=30, font 14)       │
        │   ├── label "tick=123" (TOP_MID, y=50, font 10)    │
        │   └── img "arm" (centered) [GROUP: focusable]      │
        │                                                     │
        ├── [Page: SystemInfos root] (flex column, 240×240)  │
        │   ├── item_t "Joints"                              │
        │   │   ├── icon [GROUP: focusable, 220×100]         │
        │   │   │   ├── img "joints"                         │
        │   │   │   └── label "Joints"                       │
        │   │   ├── labelInfo "J1\nJ2\nJ3\nJ4\nJ5\nJ6"     │
        │   │   └── labelData  "0\n0\n90\n0\n0\n0"          │
        │   ├── item_t "Pose6D"                              │
        │   │   └── ...                                      │
        │   ├── item_t "System"                              │
        │   │   └── ...                                      │
        │   ├── item_t "IMU"                                 │
        │   │   └── ...                                      │
        │   ├── item_t "Battery"                             │
        │   │   └── ...                                      │
        │   └── item_t "Storage"                             │
        │       └── ...                                      │
        │                                                     │
        └── [z-order: page nào được move_foreground()         │
             sẽ ở trên cùng trong page stack]                  │
                                                              │
    lv_layer_top() ──────────────────────────────────────────
        │                                                     │
        └── StatusBar container (240×22, top)                │
            ├── img "satellite" (left)                        │
            ├── label "0"                                     │
            ├── img "sd_card" (hidden)                        │
            ├── img "bluetooth" (hidden)                      │
            ├── label "REC" (hidden, right)                   │
            ├── img "battery" (right)                         │
            │   └── obj (usage bar, trắng)                    │
            └── label "100%"                                  │
```

---

## 10. Style System

Peak sử dụng **LVGL styles** với transitions:

```cpp
// SystemInfosView — Style_Init()
lv_style_init(&style.icon);
lv_style_set_width(&style.icon, 220);                    // Mặc định rộng
lv_style_set_bg_color(&style.icon, lv_color_black());
lv_style_set_text_font(&style.icon, Resource.GetFont("bahnschrift_17"));

lv_style_init(&style.focus);
lv_style_set_width(&style.focus, 70);                     // Khi focus, thu hẹp
lv_style_set_border_side(&style.focus, LV_BORDER_SIDE_RIGHT);
lv_style_set_border_width(&style.focus, 2);
lv_style_set_border_color(&style.focus, lv_color_hex(0xff0000));  // Viền đỏ

// Transition khi focus thay đổi
static lv_style_transition_dsc_t trans;
lv_style_transition_dsc_init(&trans, prop, lv_anim_path_overshoot, 200, 0, NULL);
lv_style_set_transition(&style.focus, &trans);

// Gán style cho widget
lv_obj_add_style(icon, &style.icon, 0);                  // Base style
lv_obj_add_style(icon, &style.focus, LV_STATE_FOCUSED);  // Focused override
```

Khi encoder focus vào icon:
- `LV_STATE_FOCUSED` được thêm vào widget
- `style.focus` override `style.icon`
- Width: 220 → 70 (overshoot animation 200ms)
- Viền phải màu đỏ xuất hiện
- `onFocus` callback scroll container để item vào giữa màn hình

---

## 11. Tổng kết

| Thành phần | Công nghệ | Đặc điểm |
|---|---|---|
| **GUI Library** | LVGL v8.1 | ~200KB ROM, widgets, animation, styles |
| **Screen** | 1 screen (`lv_scr_act`) | Nền đen, 240×240 |
| **Top Layer** | `lv_layer_top()` | StatusBar overlay (22px) |
| **Page** | PageBase + MVC | 3 files: Controller + View + Model |
| **Navigation** | Stack (PageManager) | Push/Pop với animation |
| **Input** | 1 device: Encoder | LV_INDEV_TYPE_ENCODER, group focus |
| **Rendering** | LvglThread (FreeRTOS) | 200Hz, flush SPI, single buffer 240×120 |
| **Styles** | LVGL style system | focus transition, overshoot path |
| **Animation** | LVGL anim + timeline | Push/Pop OVER, fade, bounce |
| **Data** | AccountSystem (Pub/Sub) | Model kéo dữ liệu qua Account::Pull() |
