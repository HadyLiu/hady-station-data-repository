# VSCode 与 Cursor 配置

> 适用：做嵌入式 C/C++ / ESP-IDF 开发。两者插件生态基本通用（Cursor 基于 VSCode）。

## 1. C/C++ 格式化

`Ctrl + ,` 打开设置，搜索以下两项：

| 设置项 | 值 | 作用 |
| --- | --- | --- |
| `C_Cpp.formatting` | `clangFormat` | 用 clang-format 而不是微软默认格式化器 |
| `Format On Save` | 勾选 | 保存即格式化，避免 diff 里混进风格改动 |

`settings.json` 里等价写法：

```json
{
  "C_Cpp.formatting": "clangFormat",
  "editor.formatOnSave": true,
  "[c]": { "editor.defaultFormatter": "ms-vscode.cpptools" },
  "[cpp]": { "editor.defaultFormatter": "ms-vscode.cpptools" }
}
```

团队协作时把 `.clang-format` 一起提交，光靠编辑器设置不同步。

## 2. 推荐插件

| 插件 | 用途 |
| --- | --- |
| `C/C++`（cpptools） | 补全、跳转、调试 |
| `clangd` | 更准的 C++ 语义补全（与 cpptools 的 IntelliSense **二选一**，同时开会导致重复提示和高 CPU） |
| `Remote - SSH` | 直连树莓派 / 服务器开发 |
| `WSL` | 在 WSL 里打开工程，工具链用 Linux 的 |
| `ESP-IDF` | ESP32 工程创建、构建、烧录、monitor |
| `Chinese (Simplified)` | 中文界面 |
| `One Dark Pro` | 主题 |

## 3. 常见组合

**在 WSL 里开发、Windows 上看界面**：装 `WSL` 插件 → 左下角「打开远程窗口」→ 选发行版 → 打开工程目录。这样编译器、调试器都是 Linux 的，路径也是 Linux 的，避免交叉工具链在 Windows 上跑不通。

**在树莓派上开发**：装 `Remote - SSH` → `Ctrl+Shift+P` → `Remote-SSH: Connect to Host` → 填 `user@192.168.1.100`。首次连接会把 VSCode Server 装到远端。

**ESP-IDF**：装插件后在命令面板执行 `ESP-IDF: Configure ESP-IDF Extension`，选「Use existing setup」指向已装好的 IDF 路径，比让插件重新下一套省几十 GB。

## 4. 坑

| 现象 | 原因 |
| --- | --- |
| 头文件全是红波浪线但能编译 | `c_cpp_properties.json` 的 `includePath` 没加 SDK 路径；跑了 CMake 的话用 `compile_commands.json` 更准 |
| 格式化把代码改乱 | 用到 `.clang-format` 与团队不一致，或 `C_Cpp.formatting` 仍是 `vcFormat` |
| 保存后整个文件都是改动 | 换行符（CRLF/LF）被格式化改动，设 `files.eol` |
| clangd 与 cpptools 抢补全 | 两个都启用了，禁用其中一个 |
| Remote-SSH 连不上 | 远端 SSH 端口/密钥问题，先用命令行 `ssh` 验证 |
| 中文注释变方块 | 字体不支持 CJK，换 `Consolas + 微软雅黑` 的组合字体或用等宽中文字体 |
