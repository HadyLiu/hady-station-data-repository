# ARM 交叉编译与 CMake

> 适用：Linux（含 WSL）上为 Cortex-M 编译裸机固件。工具链 `arm-none-eabi-gcc` + CMake + Ninja/Make。

## 1. 安装

```bash
sudo apt install -y gcc-arm-none-eabi binutils-arm-none-eabi gdb-multiarch
arm-none-eabi-gcc --version
```

`gdb-multiarch` 而不是 `gdb`：前者能识别 ARM ELF，桌面版 `gdb` 只认 x86。

## 2. C 标准怎么选

| 标准 | 参数 | 状态 | 建议 |
| --- | --- | --- | --- |
| C99 | `-std=c99` | 完整支持 | 最保守，老 SDK / 厂商 HAL 最不容易报错 |
| C11 | `-std=c11` | 完整支持 | 主流推荐，支持匿名结构体、`_Generic` |
| C17 | `-std=c17` | 完整支持 | 现代开发首选，无已知副作用 |
| C23 | `-std=c23` | 部分支持 | 尝鲜；厂商头文件常未适配，会有大量警告 |

国产 MCU 的 SDK（头文件里大量依赖 GNU 扩展、匿名 union、`__attribute__`）建议用 **`-std=gnu11`** 或 **`-std=gnu17`**，`gnu` 前缀会打开 GCC 扩展。

## 3. 标准构建流程

```bash
rm -rf build
cmake -S . -B build -G "Unix Makefiles" \
  -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake \
  -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j$(nproc)
```

- `-S . -B build`：源码目录 / 构建目录分离，构建产物不污染源码树。
- `-G "Unix Makefiles"`：没装 Ninja 时用；装了 Ninja 换成 `-G Ninja`，编译更快。
- `CMAKE_TOOLCHAIN_FILE` 必须在**首次配置**时指定；缓存已生成后改它需要删 `build/`。
- `-DCMAKE_BUILD_TYPE=Debug` 影响优化级别和调试符号，Release 下断点会乱跳。

## 4. 工具链文件要点

`cmake/gcc-arm-none-eabi.cmake` 至少包含：

```cmake
set(CMAKE_SYSTEM_NAME      Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER   arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY      arm-none-eabi-objcopy)
set(CMAKE_SIZE         arm-none-eabi-size)

# 裸机没有操作系统，不能链接标准启动文件
set(CMAKE_EXE_LINKER_FLAGS_INIT "-nostartfiles")
```

`CMAKE_SYSTEM_NAME Generic` 是关键——设成 `Linux` 会去链接 libc 的启动代码，报一堆 `_start` 未定义。

## 5. Bootloader + App 双工程结构

固件带 OTA 时拆两个子工程，共用一份 SDK：

```
<项目>/
├── arm-toolchain.cmake          # 工具链配置
├── CMakeLists.txt               # 顶层：公共编译选项 + SDK
│
├── sdk/                         # 公共 SDK，只存一份
│   ├── CMSIS/
│   └── <芯片>_StdPeriph_Driver/
│
├── bootloader/
│   ├── CMakeLists.txt
│   ├── boot_flash.ld            # 链接脚本：低端 Flash
│   ├── startup_boot.S
│   └── src/main.c               # 检测升级、跳转 App
│
└── app/
    ├── CMakeLists.txt
    ├── app_flash.ld             # 链接脚本：偏移后的 Flash（0x0 + boot 大小）
    ├── startup_app.S
    └── src/main.c
```

关键约束：

- `app_flash.ld` 的 FLASH 起始地址 = bootloader 占用大小，且 `CMakeLists` 里要生成带偏移的 `.bin`（否则 `objcopy` 出来的还是从 0 开始）。
- App 需要一份**向量表偏移**（`SCB->VTOR = APP_BASE`），否则中断进错表。
- 两个工程的 `.ld` 里 `_estack`、`RAM` 长度必须一致，只有 Flash 起点不同。

## 6. 产物查看

```bash
arm-none-eabi-size build/app.elf          # text/data/bss 占用
arm-none-eabi-objdump -h build/app.elf    # 各段地址，确认没有重叠
arm-none-eabi-nm --size-sort -S build/app.elf | tail -20   # 谁最占空间
```

## 7. 坑

| 现象 | 原因 |
| --- | --- |
| `undefined reference to '_start'` | 工具链文件里 `CMAKE_SYSTEM_NAME` 没设成 `Generic` |
| `region FLASH overflowed by N bytes` | `.ld` 里 FLASH 长度写小了，或 Debug 符号没分出去 |
| 改了 `-D` 参数不生效 | CMake 缓存了旧配置，删 `build/` 重来 |
| 编译器找不到头文件 | 用 `target_include_directories`，不要全局 `include_directories` |
| 浮点运算进了软件模拟 | Cortex-M4F 需加 `-mfpu=fpv4-sp-d16 -mfloat-abi=hard`，且目标名带 `eabihf` |
