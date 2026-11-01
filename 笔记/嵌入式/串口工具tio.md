# 串口工具 tio

> 适用：Windows / Linux / macOS。`tio` 是极简串口终端，插拔设备自动重连，比 `minicom` 省事。

## 1. Windows 安装

```powershell
mkdir C:\App\tio
cd C:\App\tio

# 下载（Windows 版由社区维护）
wget "https://github.com/konosubakonoakua/tio-win/releases/download/v3.9/tio-3.9-win.zip" -O tio.zip
Expand-Archive tio.zip -DestinationPath ./
```

没装 `wget` 的话，`Invoke-WebRequest -Uri <url> -OutFile tio.zip` 也行。

## 2. 连接

```powershell
tio COM<端口号>
```

Linux：

```bash
tio /dev/ttyUSB0
tio /dev/ttyACM0 -b 115200      # 指定波特率
```

默认 115200 8N1，多数开发板可直接用。

## 3. 常用参数

| 参数 | 作用 |
| --- | --- |
| `-b <波特率>` | 波特率，默认 115200 |
| `-l` | 列出可用串口 |
| `-n` | 不自动重连（默认会自动等设备回来） |
| `--timestamp` | 每行加时间戳，抓偶发问题很有用 |
| `--log-file <路径>` | 同时写入日志文件 |
| `-e` | 自动加载 `~/.tioconfig` 里的配置 |

## 4. 退出

`Ctrl + T` 松开再按 `q`。直接 `Ctrl + C` 在某些平台会残留占用。

## 5. 坑

| 现象 | 原因 |
| --- | --- |
| `Permission denied` | Linux 下当前用户不在 `dialout` 组：`sudo usermod -aG dialout $USER` |
| 打不开端口，被其他程序占用 | 串口助手 / IDE 的 monitor 已占；先关掉 |
| 输出乱码 | 波特率不匹配；ESP32 的 ROM log 默认 115200 |
| 设备一插拔就退出 | 去掉 `-n`，让它自动重连 |
| WSL 里找不到 `/dev/ttyUSB0` | 需要先 `usbipd attach`，见 [WSL 绑定 USB 设备](../Windows/WSL绑定USB设备.md) |
