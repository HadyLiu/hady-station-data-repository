# Ventoy 制作多合一启动盘

> 适用：Windows / Linux。一次装好 Ventoy，之后往 U 盘里**丢 ISO 文件**即可，不用反复格式化。

## 1. 原理

Ventoy 在 U 盘上建一个引导程序 + 一个普通分区。开机时它扫描分区里的 `.iso / .wim / .img / .vhdx`，列成菜单供选择。ISO 原样存放，不破坏文件。

## 2. 安装

1. 下载：`https://github.com/ventoy/Ventoy/releases` → `ventoy-x.x.xx-windows.zip`
2. 解压，管理员运行 `Ventoy2Disk.exe`
3. 选中目标 U 盘（**确认盘符，装错会清空整盘**）
4. 可选：分区类型 `GPT`（新机器 / UEFI）或 `MBR`（老机器兼容更好）
5. 点「安装」，两次确认后等待完成

## 3. 使用

把 ISO 直接拷到 U 盘分区根目录（或任意子目录，Ventoy 会递归扫描）。开机按启动热键（F12 / F11 / Esc / F8，视主板而定）选 U 盘，出现 Ventoy 菜单后选 ISO 回车。

## 4. 常用维护

| 操作 | 说明 |
| --- | --- |
| 升级 | 新版 Ventoy2Disk 点「升级」，**保留数据**，不动已有 ISO |
| 扩容分区 | Ventoy2Disk → 配置 → 分区设置，改「保留空间」后需重新分配 |
| 启动自定义 | U 盘根目录放 `ventoy/ventoy.json`，可设默认项、菜单主题、自动安装脚本 |
| 安全启动 | BIOS 开了 Secure Boot 时，首次启动需按提示「Enroll key」注册 Ventoy 证书 |

## 5. 坑

| 现象 | 原因 |
| --- | --- |
| 装完 U 盘容量变小 | 正常，Ventoy 预留了 32MB 引导分区 |
| ISO 里没有启动菜单项 | 该 ISO 不被支持（部分 Windows PE 定制版），换官方镜像验证 |
| Secure Boot 下报 `Verification failed` | 未注册 MOK，按提示 enroll 或临时关 Secure Boot |
| 拷大 ISO 中途失败 | U 盘是 FAT32，单文件上限 4GB；装 Ventoy 时选 exFAT/NTFS 分区 |
