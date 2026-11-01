# Ubuntu 常用操作

> 适用：Ubuntu 22.04 / 24.04 桌面版。命令均需普通用户 + `sudo`。

## 1. 输入法切换快捷键

`设置 → 键盘 → 查看及自定义快捷键 → 打字 → 切换上个输入法`

默认 `Super + Space` 常与系统快捷键冲突，改成 `Ctrl + Space` 更稳。

## 2. 桌面不显示其它磁盘

`设置 → Ubuntu 桌面 → Configure Dock Behavior → Dock → 关闭「显示卷和设备」`

## 3. 包管理

```bash
# 刷新索引
sudo apt update
# 升级已安装包
sudo apt upgrade
sudo apt full-upgrade     # 允许因依赖变化卸载包，跨版本升级时用
# 清理
sudo apt autoremove      # 删除不再被依赖的包
sudo apt autoclean       # 清空过期 deb 缓存

# 查找
apt search <关键字>
apt show <包名>
```

```bash
sudo apt install <包名>
sudo apt remove <包名>    # 保留配置文件
sudo apt purge <包名>     # 连配置一起删
```

## 4. 大版本升级

```bash
sudo nano /etc/update-manager/release-upgrades   # Prompt=lts
sudo do-release-upgrade
```

升级前先 `sudo apt update && sudo apt full-upgrade`，并确认有可用回滚手段（快照/备份）。

## 5. 双系统时间差 8 小时

Windows 把 RTC 当本地时间，Linux 当 UTC，导致来回差 8 小时。让 Linux 也用本地时间：

```bash
timedatectl status
sudo timedatectl set-local-rtc 1 --adjust-system-clock
```

（反过来把 Windows 改成 UTC 更"正确"，但要改注册表，日常用这条更省事。）

## 6. 隐藏文件夹

GNOME 的文件管理器额外读 `~/.hidden`——每行一个名字：

```bash
echo "snap" >> ~/.hidden
```

注意：只影响文件管理器显示，不影响 `ls`（`ls` 靠文件名前的 `.`）。

## 7. 脚本里的路径变量

```bash
pwd          # 交互时看当前目录
echo "$PWD"  # 脚本中用变量
echo "$HOME" # 用户主目录
```

脚本里不要写死 `/home/<user>`，用 `$HOME`；也不要用 `cd` 后的相对路径，统一 `"${PWD}"` / 绝对路径。

## 8. 打包与解包

```bash
tar -cvf backup.tar mydir        # 打包
tar -xvf backup.tar              # 解包
tar -czvf backup.tar.gz mydir    # 打包 + gzip
tar -xzvf backup.tar.gz          # 解压 gzip
```

`-f` 后面必须紧跟文件名，顺序不能乱。

## 9. 生成随机密码

```bash
# 15 字节 → base64 后约 20 字符，取前 N 位
openssl rand -base64 24 | tr -d '\n' | head -c 31; echo
```

`base64` 输出可能含 `+ / =`，写进配置文件（YAML、ini）时容易被转义，需要纯字母数字就加 `| tr -dc 'A-Za-z0-9'`。

## 10. 查看系统版本

```bash
lsb_release -a
cat /etc/os-release
```

## 11. 删除文件

```bash
rm -f file          # 单文件
rm -rf dir          # 递归 + 强制，无确认、不可恢复
```

`rm -rf` 没有回收站。删目录前先 `ls` 确认路径，或用 `gio trash` 走回收站。
