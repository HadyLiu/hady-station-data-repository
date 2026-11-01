# WSL 安装与图形界面

> 适用：Windows 10 2004+ / Windows 11，WSL2 + WSLg。本文含「在 WSL 里跑 Linux 版 IDE」的完整配置。

## 1. 安装

管理员 PowerShell：

```powershell
wsl --install                       # 默认装 Ubuntu + 开启虚拟机平台
wsl --list --online                 # 看可装发行版
wsl --install -d Ubuntu-24.04
wsl --update                        # 更新内核（WSLg 依赖它）
```

装完需重启。检查

```powershell
wsl -l -v      # 确认 VERSION 是 2，不是 1
```

## 2. 备份 / 迁移 / 恢复（必做）

WSL 的 rootfs 在 `%LOCALAPPDATA%` 里，重装系统会丢。定期导出：

```powershell
# 导出（先 wsl --shutdown，否则可能导出失败）
wsl --shutdown
wsl --export Ubuntu D:\backup\ubuntu-20260101.tar

# 导入到新位置
wsl --import Ubuntu D:\WSL\Ubuntu D:\backup\ubuntu-20260101.tar

# 注销旧实例（导入成功后）
wsl --unregister Ubuntu
```

`--export` 出来的是完整文件系统，可压缩体积会小很多。

## 3. 图形界面

WSL2 自带 WSLg（Windows 11 / Win10 21H2+），X11 应用直接能开窗口，**无需装 X Server**。先装依赖：

```bash
sudo apt update
sudo apt install -y x11-apps
xeyes                # 能弹出窗口说明 WSLg 正常
```

若没有窗口：确认 `wsl --update` 已执行，且 `echo $DISPLAY` 有值（通常 `:0`）。

## 4. 安装 Linux 版 GUI 软件（以 Simplicity Studio 为例）

这类 IDE 依赖一堆 Chromium 运行时库，缺一个就白屏或闪退：

```bash
# 基础运行库
sudo apt install -y libasound2t64 libnss3 libatk1.0-0t64 libatk-bridge2.0-0t64 \
  libcups2t64 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 \
  libxrandr2 libgbm1 libpango-1.0-0 libcairo2 libsecret-1-0

# 工具链常用
sudo apt install -y bzip2 pbzip2 tar unzip zip xz-utils
```

安装包（本机下载后 `dpkg -i` 或直接 `apt install ./xxx.deb`），启动：

```bash
/opt/SimplicityInstaller/simplicityinstaller --no-sandbox
```

`--no-sandbox` 在 WSL 下基本必须，否则 Chromium 内核因权限模型报错退出。

## 5. 从 Windows 一键启动 WSL 里的 GUI 程序

新建快捷方式（`.lnk`），目标填：

```text
"C:\Program Files\WSL\wslg.exe" -d Ubuntu --cd "~" -- /opt/SimplicityInstaller/simplicityinstaller --no-sandbox
```

- `wslg.exe`：带图形支持启动，无黑窗。
- `-d Ubuntu`：发行版名，`wsl -l -v` 查。
- `--cd "~"`：工作目录设成家目录，很多 IDE 用它存配置。
- 路径含空格/变量时必须整体加引号。

用 `wsl.exe` 的版本会带一个控制台黑框，仅在需要看 stdout 时用：

```text
C:\Windows\System32\wsl.exe -d Ubuntu -u <user> /opt/SimplicityInstaller/simplicityinstaller --no-sandbox
```

## 6. 坑

| 现象 | 原因 / 解决 |
| --- | --- |
| 导出时报 `The process cannot access the file` | 没先 `wsl --shutdown`，有进程占用 vhdx |
| GUI 程序白屏 | 缺 `libgbm1` / `libnss3` 等 Chromium 依赖 |
| 启动即退、无报错 | 加 `--no-sandbox`，或在终端里跑看 stderr |
| 窗口模糊 | 高分屏缩放问题，在 Windows 端给 `.exe` 设「替代高 DPI 缩放行为 → 应用程序」 |
| `wsl --import` 后默认用户变 root | 导入不保留默认用户，需在 `/etc/wsl.conf` 写 `[user] default=<user>` |
