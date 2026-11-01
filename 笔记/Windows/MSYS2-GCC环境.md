# MSYS2 搭建 GCC 环境（Windows）

> 适用：Windows。用 MSYS2 的 UCRT64 环境获得现代 GCC/G++/GDB，可用于桌面端 C/C++ 开发。

## 1. 为什么选 UCRT64

MSYS2 有三个环境，**只建议 UCRT64**：

| 环境 | 运行时 | 说明 |
| --- | --- | --- |
| **UCRT64** | 微软 UCRT | 官方推荐，与 Windows 兼容性最好，第三方库支持全 |
| MINGW64 | msvcrt | 老环境，部分库只在这有 |
| MSYS | cygwin | POSIX 模拟层，**不要**用它编原生 exe |

## 2. 更新与安装

从开始菜单打开 **MSYS2 UCRT64**（不是 MSYS2 MSYS）：

```bash
pacman -Syu
```

提示关闭终端时，关掉后重开再跑一次 `pacman -Syu`。

装工具链（二选一）：

```bash
# 推荐：完整工具链，含 gcc / g++ / gdb / make / cmake
pacman -S mingw-w64-ucrt-x86_64-toolchain

# 精简：只要编译器
pacman -S mingw-w64-ucrt-x86_64-gcc
```

`toolchain` 会列出组件清单，直接回车（全选）再 `y`。

验证：

```bash
gcc --version
g++ --version
```

## 3. 在 PowerShell / VS Code 里用 gcc

MSYS2 里的 `gcc` 只在 MSYS2 终端可用。要在别处用，把 bin 目录加进 PATH：

1. `Win + R` → `sysdm.cpl` → 高级 → 环境变量
2. 选中 `Path` → 编辑 → 新建，填 UCRT64 的 bin（默认安装路径）：

```text
C:\App\DevelopmentPc\MSYS2\ucrt64\bin
```

3. 一路确定，然后**新开**一个终端验证 `gcc --version`。

只加 `ucrt64\bin`，不要再加 `usr\bin`——后者会引入 MSYS 的 POSIX 工具，导致路径混用问题。

## 4. 补齐 exe 的 DLL 依赖

Windows 上把 `gcc` 编出来的 exe 拷给别人经常报「缺少 libstdc++-6.dll」「缺少 libgcc_s_seh-1.dll」。原因是这些库在 `ucrt64\bin` 里，没跟着 exe 走。

`ldd` 是 Linux/MSYS2 命令，**PowerShell / CMD 里没有**。三种做法：

### 方案 A：装 ntldd（PowerShell 也能查）

```bash
pacman -S mingw-w64-ucrt-x86_64-ntldd
```

重开 PowerShell：

```powershell
ntldd VDR.exe
```

### 方案 B：一键把依赖拷到 exe 旁边

```powershell
$exe    = ".\build\app.exe"
$outDir = ".\build\"

ntldd -R $exe | Select-String "ucrt64" | ForEach-Object {
    $dllPath = ($_ -split '=>')[1].Trim() -split ' ' | Select-Object -First 1
    if (Test-Path $dllPath) {
        Write-Host "复制: $dllPath" -ForegroundColor Green
        Copy-Item $dllPath -Destination $outDir -Force
    }
}
```

只筛 `ucrt64` 是因为系统 DLL（kernel32、msvcrt）目标机自带，拷过去反而可能冲突。

### 方案 C：直接用 MSYS2 终端

```bash
cd /c/项目路径        # 注意是 /c/ 而不是 C:\
ldd build/app.exe
```

## 5. 坑

| 现象 | 原因 |
| --- | --- |
| `pacman` 报数据库锁定 | 上一次 `pacman` 被强制中断，删 `/var/lib/pacman/db.lck` |
| 编出的 exe 在别人机器缺 DLL | 见第 4 节，静态链接可加 `-static -static-libgcc -static-libstdc++` |
| gcc 能用但 gdb 起不来 | `toolchain` 未装全，单独 `pacman -S mingw-w64-ucrt-x86_64-gdb` |
| 路径斜杠混乱 | MSYS2 里 `C:\a\b` 写作 `/c/a/b`；给原生程序传参时才用反斜杠 |
