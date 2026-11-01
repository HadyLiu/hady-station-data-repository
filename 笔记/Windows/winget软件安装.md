# winget 软件安装

> 适用：Windows 10 1809+ / Windows 11。`winget` 是微软官方包管理器，命令行装软件不用点下一步。

## 1. winget 缺失或损坏时修复

以**管理员**身份打开 PowerShell：

```powershell
Install-Module -Name Microsoft.WinGet.Client -Force -Scope CurrentUser
Repair-WinGetPackageManager -AllUsers
```

`Repair-WinGetPackageManager` 会自动补齐 App Installer 及其依赖，比手动去 Store 装可靠。

验证：

```powershell
winget --version
```

## 2. 日常用法

```powershell
winget source update                 # 刷新软件源索引（找不到包时先跑这个）
winget search <关键字>
winget install <包ID>
winget install --exact <包ID>        # 精确匹配，避免装错同名包
winget upgrade --all                 # 升级全部
winget list                          # 已安装列表
```

包 ID 形如 `Publisher.Package`（如 `Rustlang.Rustup`），也可以用 Store ID（如 `9WZDNCRFHVN5` 是新版计算器）。

## 3. 常见包的 ID

| 软件 | 命令 |
| --- | --- |
| Rust | `winget install Rustlang.Rustup` |
| usbipd-win（WSL USB 直通） | `winget install --interactive --exact dorssel.usbipd-win` |
| TortoiseGit | `winget install TortoiseGit.TortoiseGit` |

## 4. 坑

| 现象 | 原因 |
| --- | --- |
| `无法将"winget"识别为...` | 未装 App Installer，或 PATH 未刷新（重开终端） |
| `找不到与输入条件匹配的程序包` | 源索引过期，先 `winget source update` |
| 安装需要交互（如驱动） | 加 `--interactive` 让安装器弹 GUI |
| 双击 exe 提示「无法运行 / 已被阻止」 | 文件来自网络被加了「锁定」标记：右键 → 属性 → 勾选「解除锁定」→ 确定 |
