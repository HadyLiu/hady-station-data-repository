# ESP32 烧录与 ESP-IDF

> 适用：ESP32 / ESP32-C3 / ESP32-S3。命令行擦除、烧录，以及用 eim 装 IDF。

## 1. esptool

```bash
pip install esptool          # 或下载独立 exe：github.com/espressif/esptool
```

### 擦除

```powershell
esptool --chip esp32c3 -b 2000000 --before usb_reset --after hard_reset --port COM<端口号> erase_flash
```

### 烧录

```powershell
esptool --chip esp32c3 -b 2000000 --before usb_reset --after hard_reset --port COM<端口号> write_flash --flash_mode dio --flash_size 4MB --flash_freq 80m 0x0 bootloader.bin 0x8000 partition-table.bin 0x10000 app.bin
```

用外部 USB 转串口（非原生 USB）时把 `--before usb_reset` 换成 `--before default_reset`：

```powershell
esptool --chip esp32c3 -b 2000000 --before default_reset --after hard_reset --port COM<端口号> write_flash --flash_mode qio --flash_size 4MB --flash_freq 80m 0x0 bootloader.bin 0x8000 partition-table.bin 0x10000 app.bin
```

地址不能随便改：`0x0` bootloader、`0x8000` 分区表、`0x10000` 应用（带 OTA 时 `0x20000`）。`flash_mode` 用 `dio` 还是 `qio` 取决于模组封装，写错会启动失败。

## 2. 安装 ESP-IDF（eim）

Linux 依赖：

```bash
sudo apt update && sudo apt install -y flex bison gperf ccache dfu-util cmake \
  libffi-dev libusb-1.0-0 libssl-dev python3-pip python3-venv
```

拉取并用向导安装：

```bash
/home/<user>/.espressif/eim/eim wizard --idf-features ide
```

向导里选：

- 目标芯片（决定装哪套工具链，选全了体积很大）
- IDF 版本
- 功能项：`ide`（含 VS Code 插件支持）、`docs`、`examples`

装完按提示 `source export.sh`，或让它写进 shell rc。

## 3. 常用命令

```bash
idf.py set-target esp32c3
idf.py menuconfig
idf.py build
idf.py -p /dev/ttyACM0 flash monitor      # 烧录并开串口监视
idf.py fullclean                          # 清构建目录（换 target 后必须要）
```

`Ctrl + ]` 退出 monitor。

## 4. LCD 引脚配置示例

SPI 屏接 ESP32-C3 的一段配置（按原理图改）：

```c
#define EXAMPLE_LCD_PIXEL_CLOCK_HZ     (40 * 1000 * 1000)
#define EXAMPLE_LCD_BK_LIGHT_ON_LEVEL  1
#define EXAMPLE_LCD_BK_LIGHT_OFF_LEVEL !EXAMPLE_LCD_BK_LIGHT_ON_LEVEL

/* 引脚映射（依据原理图） */
#define EXAMPLE_PIN_NUM_SCLK           14    // LCD_SCL → IO14
#define EXAMPLE_PIN_NUM_MOSI           13    // LCD_SDA → IO13
#define EXAMPLE_PIN_NUM_MISO           -1    // 无读数据线
#define EXAMPLE_PIN_NUM_LCD_DC         21    // LCD_RS  → IO21
#define EXAMPLE_PIN_NUM_LCD_RST        22    // LCD_REST → IO22
#define EXAMPLE_PIN_NUM_LCD_CS         15    // LCD_CS  → IO15
#define EXAMPLE_PIN_NUM_BK_LIGHT       23    // LCD_LED → IO23
#define EXAMPLE_PIN_NUM_TOUCH_CS       -1    // 无触摸片选
```

注意 ESP32-C3 的 IO11~IO17 常被内部 Flash 占用，用之前查芯片数据手册的「strapping / SPI flash 引脚」章节。

## 5. 坑

| 现象 | 原因 |
| --- | --- |
| `Failed to connect: Timed out waiting for packet header` | 没进下载模式；按住 BOOT 再点 RST，或换 `--before` 模式 |
| 烧完串口无输出 | `flash_mode` 或 `flash_size` 与实际模组不符 |
| `A fatal error occurred: MD5 of file does not match` | 波特率过高，降到 `-b 460800` |
| 应用跑不起来、反复重启 | 分区表地址与 `partitions.csv` 不匹配，或 `0x10000` 处不是有效镜像 |
| `idf.py` 找不到命令 | 没 `source export.sh`，新开终端需重新 source |
| 换芯片后编译报错 | 没 `idf.py fullclean`，旧 target 的 CMake 缓存还在 |
