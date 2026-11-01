# pyOCD 烧录与调试

> 适用：Linux（含 WSL）。用 pyOCD 驱动 ST-LINK / CMSIS-DAP 探针烧录 ARM Cortex-M 固件，替代 OpenOCD。

## 1. 安装

用独立 venv 隔离，避免污染系统 Python：

```bash
python3 -m venv tools/pyocd-venv
tools/pyocd-venv/bin/pip install -U pyocd

# 查看连上的探针
tools/pyocd-venv/bin/pyocd list
```

或者从 GitHub 下 release：`https://github.com/pyocd/pyOCD`

## 2. 权限：udev 规则（一劳永逸）

普通用户访问 USB 探针会被拒（`Permission denied` / 设备未列出）。每次 `sudo` 不是办法，写规则：

```bash
sudo nano /etc/udev/rules.d/49-stlink.rules
```

```ini
# ST-LINK/V2
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", MODE:="0666"
# ST-LINK/V2.1
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", MODE:="0666"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3752", MODE:="0666"
# ST-LINK/V3
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374d", MODE:="0666"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374e", MODE:="0666"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374f", MODE:="0666"
```

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

重插探针生效。`MODE:="0666"` 里的 `:=` 是强制赋值，不能写成 `=`（否则会被系统默认规则覆盖）。

WSL 下还要先把设备透传进来，见 [WSL 绑定 USB 设备](../Windows/WSL绑定USB设备.md)。

## 3. 烧录

```bash
tools/pyocd-venv/bin/pyocd flash \
  --pack <芯片支持包>.pack \
  -t <目标型号> \
  --connect under-reset \
  -f 4M \
  -a 0x0 \
  build/full.bin
```

| 参数 | 含义 |
| --- | --- |
| `--pack` | DFP 支持包，提供 Flash 算法；国产芯片大多要自己下 |
| `-t` | target 名，必须与 pack 里的名字完全一致 |
| `--connect under-reset` | 复位期间连接，防止程序跑飞或低功耗睡死连不上 |
| `-f` | Flash 大小，用于校验越界 |
| `-a` | 烧录起始地址；App 分区要改成偏移地址 |

## 4. 调试

```bash
# 起 GDB server
tools/pyocd-venv/bin/pyocd gdbserver -t <型号> --pack <包>.pack

# 另一个终端
gdb-multiarch build/app.elf
(gdb) target extended-remote :3333
(gdb) load
(gdb) monitor reset halt
(gdb) break main
(gdb) continue
```

## 5. 擦除 / 复位

```bash
tools/pyocd-venv/bin/pyocd erase -t <型号> --chip          # 全片擦除
tools/pyocd-venv/bin/pyocd erase -t <型号> -a 0x8000 -s 0x1000  # 按扇区
tools/pyocd-venv/bin/pyocd reset -t <型号>
```

## 6. 坑

| 现象 | 原因 |
| --- | --- |
| `pyocd list` 空 | udev 规则没生效，或 WSL 未 attach 设备 |
| `no target type matches` | `-t` 与 pack 内名称不一致，用 `pyocd list --targets --pack <包>` 查 |
| `Error: target not halted` | 加 `--connect under-reset`，或先 `pyocd reset -t ...` |
| 烧完不运行 | 缺 `--after hard_reset`，或用 `pyocd reset` 手动复位 |
| `.pack` 里的算法与实物 Flash 不符 | 型号后缀（容量）选错，例如 K8 与 K6 算法不同 |
