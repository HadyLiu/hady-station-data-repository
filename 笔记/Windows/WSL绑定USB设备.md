# WSL 绑定 USB 设备

> 适用：WSL2。用 `usbipd-win` 把 Windows 上的 USB 设备（调试器、串口、烧录器）透传给 WSL 里的 Linux 工具链。

## 1. 安装

管理员 PowerShell：

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

`--interactive` 是必须的，安装器需要弹 UAC/配置界面。

## 2. 三步连接

**① 列出设备**（管理员 PowerShell）

```powershell
usbipd list
```

记下目标设备的 `BUSID`（形如 `4-4`）和当前 `STATE`（应为 `Not shared`）。

**② 绑定到 WSL**（管理员，只需做一次）

```powershell
usbipd bind --busid 4-4
```

再 `usbipd list` 确认 `STATE` 变成 `Shared`。绑定是**持久**的，重启后仍是 Shared。

**③ 附加到 WSL2**（普通权限即可，每次开机/插拔后要重做）

```powershell
usbipd attach --wsl --busid 4-4
```

附加后 Windows 侧该设备不可用，直到 detach。

## 3. WSL 内确认与授权

```bash
sudo apt install -y usbutils
lsusb                       # 能看到设备即成功
```

**串口权限**（否则 `Permission denied /dev/ttyUSB0`）：

```bash
sudo usermod -aG dialout $USER
newgrp dialout              # 当前会话生效，免重启 WSL
sudo chmod 666 /dev/ttyUSB0 # 临时放开，重插后失效
```

**调试器权限**（ST-LINK / J-Link / CMSIS-DAP）：每次插拔后设备节点权限可能重置，用 udev 规则一劳永逸。示例见 [pyOCD 烧录](../嵌入式/pyOCD烧录.md)。

## 4. 断开

```powershell
usbipd detach --busid 4-4
```

或者直接拔插——拔插后设备回到 Windows，需要重新 `attach`。

## 5. 坑

| 现象 | 原因 |
| --- | --- |
| `usbipd: command not found` | 装完没重开终端，PATH 未刷新 |
| `bind` 报 `Access denied` | 没用管理员 PowerShell |
| `attach` 报 `device is not shared` | 漏了第 ② 步，或设备被重新枚举后 BUSID 变了 |
| WSL 里 `lsusb` 看不到 | WSL 内核版本过旧（`wsl --update`），或发行版是 WSL1（`wsl -l -v` 确认是 2） |
| 传到 WSL 后 Windows 端软件报设备丢失 | 正常现象，同一设备同一时刻只能一边用 |
