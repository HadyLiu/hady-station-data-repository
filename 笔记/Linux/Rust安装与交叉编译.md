# Rust 安装与交叉编译

> 适用：Ubuntu / Windows。嵌入式 Rust 走「rustup + target + probe-rs」路线，不需要另装 ARM 工具链。

## 1. 安装

Linux：

```bash
sudo apt update
sudo apt install -y build-essential curl pkg-config libssl-dev

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

rustup default stable
rustup component add rustfmt clippy rust-analyzer

rustc --version && cargo --version
```

Windows（PowerShell）：

```powershell
winget install Rustlang.Rustup
```

Windows 还需要 MSVC 生成工具（勾选「使用 C++ 的桌面开发」），否则链接阶段报缺 `link.exe`。

## 2. 升级

```bash
rustup update
```

## 3. 交叉编译到 ARM Cortex-M

`rustc` 自带 LLVM，加 target 就能编 ARM，链接器用自带的 `rust-lld`——**不需要** `gcc-arm-none-eabi`。

```bash
rustup target add thumbv6m-none-eabi          # Cortex-M0 / M0+
rustup target add thumbv7m-none-eabi          # Cortex-M3
rustup target add thumbv7em-none-eabihf       # Cortex-M4F / M7F（带 FPU）
rustup target add thumbv8m.main-none-eabihf   # Cortex-M33
```

`eabihf` 表示硬浮点 ABI，选错会链接报 `uses VFP register arguments` 之类错误。

## 4. 嵌入式常用工具

```bash
cargo install probe-rs-tools flip-link cargo-binutils
rustup component add llvm-tools
```

| 工具 | 作用 |
| --- | --- |
| `probe-rs` | 烧录 + 调试，替代 OpenOCD + GDB 组合 |
| `flip-link` | 把栈放到 RAM 顶部的独立区域，溢出时立刻 HardFault 而不是静默踩数据 |
| `cargo-binutils` | `cargo size` 看固件占用、`cargo objdump` 反汇编 |

## 5. 什么时候才需要 ARM GCC

只有要**编译厂商提供的 C 库**（HAL、DSP、闭源 `.a`）时才需要，因为那部分 C 代码得用 GCC 编：

```bash
sudo apt install -y gcc-arm-none-eabi
```

纯 Rust 工程不用装。

## 6. Cargo 基本操作

```bash
cargo new hello            # 建工程（默认带 git）
cargo build                # debug 构建
cargo run                  # 构建并运行
cargo check                # 只做类型检查，比 build 快得多，改代码后先用它
cargo build --release      # 优化构建
cargo build --target thumbv7m-none-eabi   # 交叉编译
```

直接用 `rustc` 编单文件也可以（`rustc main.rs && ./main`），但一旦有依赖就得回 Cargo。

## 7. 坑

| 现象 | 原因 |
| --- | --- |
| `error: linker 'link.exe' not found` | Windows 缺 MSVC Build Tools |
| `note: run with rustup target add ...` | 忘了加 target |
| `error: no global memory allocator found` | `no_std` 工程用了 `Vec`/`String`，需自己实现 allocator 或改用 `heapless` |
| `rustup: command not found` | 没 `source "$HOME/.cargo/env"`，或新 shell 未加载 |
