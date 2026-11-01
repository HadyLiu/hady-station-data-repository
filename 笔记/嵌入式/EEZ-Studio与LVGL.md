# EEZ Studio 与 LVGL

> 适用：ESP-IDF + LVGL 9.x 项目。用 EEZ Studio 拖界面，生成 C 代码，工程里调用 `ui_init()` / `ui_tick()`。

## 1. 在 EEZ Studio 里设计界面

1. 打开工程文件 `packages/main/main.eez-project`
2. 工程参数（必须与固件一致）：
   - 类型 **LVGL**
   - 版本 **9.3.0**（与 `idf_component.yml` 里的 `lvgl` 版本一致）
   - 分辨率 **480 × 320**（横屏）
3. 改完界面点 **Build / Generate**，代码导出到 `packages/main/src/ui/`：

```
packages/main/src/ui/
├── ui.c / ui.h            ← 入口：ui_init()、ui_tick()
├── screens.c / screens.h  ← 界面与 objects
├── actions.h              ← 事件声明
├── styles.c / images.c / fonts.h
```

4. 按钮点击：在 EEZ 里绑 **Action**，生成后在 `actions.h` 里有函数声明，自己在 `.c` 里实现。

## 2. 接进 ESP-IDF

### ① 把生成代码加入编译

`main/CMakeLists.txt`：

```cmake
idf_component_register(
    SRCS "main.c"
         "packages/main/src/ui/ui.c"
         "packages/main/src/ui/screens.c"
         "packages/main/src/ui/styles.c"
         "packages/main/src/ui/images.c"
    INCLUDE_DIRS "." "packages/main/src/ui"
)
```

或者把 `components/ui/CMakeLists.txt` 写成 GLOB 自动收集 `.c`，避免每次新增界面都改文件。

### ② 启动 UI

LVGL 初始化、display、input 都就绪后，**持有 LVGL 锁**再调用：

```c
#include "ui.h"

ui_init();          // 建主题、创建所有 Screen、加载默认屏
```

原来默认示例的 `example_lvgl_demo_ui(display)` 要注释掉，两者会抢同一个 display。

### ③ 主循环里周期 tick

在 LVGL 任务里 `lv_timer_handler()` 前后加锁，并调用：

```c
ui_tick();          // 动画、定时器、脏区刷新
```

**只调 `ui_init()` 不调 `ui_tick()`，界面会停在第一帧**——这是最常见的错。

### ④ 头文件包含

生成的 `ui.h` 里是 `#include <lvgl/lvgl.h>`，与 IDF 组件的 include 方式一致，只要把 `packages/main/src/ui` 加进 `INCLUDE_DIRS` 即可。

## 3. API 速查

| 函数 / 对象 | 作用 |
| --- | --- |
| `ui_init()` | 建主题、创建全部 Screen、加载默认屏 |
| `ui_tick()` | 每轮 LVGL 任务调用，驱动界面更新 |
| `loadScreen(SCREEN_ID_MAIN)` | 切屏（多屏时用） |
| `objects.main` | 主屏根对象（见 `screens.h` 的 `objects_t`） |

## 4. 分辨率必须对齐

| 来源 | 分辨率 |
| --- | --- |
| EEZ 工程 | 480 × 320（横屏） |
| `main.c` 里 LCD | 320 × 480（竖屏） |

不一致的后果：控件错位、只显示一部分、触摸坐标整体偏移。两种修法：

- 在 EEZ 里把工程改成 320 × 480 再 Generate（推荐，UI 设计时就能看准）
- 或固件里旋转显示（`lv_display_set_rotation`），让物理方向与设计一致

## 5. 触摸

`main.c` 里已把触摸注册为 LVGL **POINTER** 输入设备，EEZ 生成的 Button/Slider 会自动响应，UI 层不需要再读 I2C。

坐标系问题：若旋转过显示，触摸的 `x/y` 也要跟着做一次映射，否则点击位置和视觉位置对不上。

## 6. 日常流程

```
EEZ Studio 改界面
   ↓ Generate → packages/main/src/ui/
   ↓
idf.py build flash monitor
   ↓
串口看显示 + 试触摸
```

只改了 `src/ui/` 下的生成文件，不需要动 `.eez-project`；反之改了 `.eez-project` 必须在 EEZ 里重新 Generate，否则代码被覆盖或不同步。

## 7. 坑

| 现象 | 原因 |
| --- | --- |
| 黑屏，无报错 | 只调了 `ui_init()`，漏了 `ui_tick()` |
| 界面闪烁 / 花屏 | 调 UI API 时没持 LVGL 锁（`lvgl_port_lock()`） |
| 编译报 `lv_obj_t` 未定义 | LVGL 版本与 EEZ 生成代码不匹配（9.x 与 8.x API 差异大） |
| 界面显示不全 | EEZ 分辨率与 LCD 不一致 |
| 改界面后代码被还原 | 在 EEZ 里没 Generate，直接手改生成文件 |
| 中文字符显示成方块 | 字体没包含对应字形，EEZ 里要显式选 CJK 字体并生成 |
