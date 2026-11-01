# 树莓派 SSH：开启与加固

> 适用：Raspberry Pi 4 + Ubuntu Server（U 盘启动），串口控制台 115200 兜底。
> 示例：主机 `<user>@<host>`，Pi 用户 `ubuntu`，地址 `192.168.1.100`。

## 1. 安装并启用

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh

# 确认在监听
sudo ss -tlnp | grep -E '22|2222'
```

```bash
ssh ubuntu@192.168.1.100
```

Ubuntu Server 首次启动可能需要手动装 `openssh-server`（Pi 官方 Raspberry Pi OS 默认已开）。

## 2. 密钥登录（正确姿势）

**密钥在「客户端」生成，公钥写进 Pi 的 `authorized_keys`。** 反过来在 Pi 上 `ssh-keygen` 再指望用电脑登进去，是行不通的——那是 Pi 自己的密钥对。

客户端：

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_pi -C "pi"
cat ~/.ssh/id_rsa_pi.pub
```

写入 Pi，方式 A（密码还开着时最省事）：

```bash
ssh-copy-id -i ~/.ssh/id_rsa_pi.pub ubuntu@192.168.1.100
```

方式 B（只能走串口）：

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys      # 粘贴客户端 id_rsa_pi.pub 整行
chmod 600 ~/.ssh/authorized_keys
```

验证：

```bash
ssh -i ~/.ssh/id_rsa_pi ubuntu@192.168.1.100
ls -la ~/.ssh/authorized_keys    # 大小为 0 说明没写进去
```

## 3. 加固

**顺序很重要：先确认密钥能登，再关密码；全程留着串口做兜底。**

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config.d/99-harden.conf
```

```ini
Port 2222
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers ubuntu
MaxAuthTries 3
LoginGraceTime 30
```

```bash
sudo sshd -t && sudo systemctl reload ssh
```

`sshd -t` 必须先过——配置有语法错时 reload 会让 SSH 直接断掉。

改端口后**新开一个窗口**测试（别关当前会话）：

```bash
ssh -i ~/.ssh/id_rsa_pi -p 2222 ubuntu@192.168.1.100
```

防火墙：

```bash
sudo ufw allow 2222/tcp
sudo ufw delete allow 22/tcp     # 确认 2222 可用后再删
```

## 4. 故障排查

### `Permission denied (publickey)`

服务端只认公钥，而客户端公钥没进 `authorized_keys`。串口临时开回密码登录：

```bash
echo 'PasswordAuthentication yes' | sudo tee /etc/ssh/sshd_config.d/60-password.conf
sudo sshd -t && sudo systemctl reload ssh
```

装好公钥后再把 `60-password.conf` 删掉。

### 自检命令

```bash
sudo systemctl status ssh --no-pager
sudo ss -tlnp | grep -E '22|2222'
ip a show wlan0
sudo sshd -t
```

### 看客户端到底卡在哪

```bash
ssh -v -p 2222 ubuntu@192.168.1.100
```

看末尾：

| 末尾信息 | 含义 |
| --- | --- |
| `Connection refused` | 端口不通 / 服务没起 / 防火墙拦了 |
| `Connection timed out` | 网络不通 / IP 变了 / 路由问题 |
| `Permission denied` | 网络通了，认证失败 |

### 踩坑记录

| 现象 | 原因 |
| --- | --- |
| 客户端 `Permission denied (publickey)` | 密码已关 + `authorized_keys` 为空 |
| Pi 上已有 `id_rsa` 密钥对仍登不上 | 密钥生成在 Pi 上，客户端没有对应私钥 |
| sshd 日志显示某 IP 认证后立即断开 | 连上了但认证失败，多为公钥不匹配 |
| 加固后彻底连不上 | 改成 2222 却仍连 22；或关密码前密钥没通 |
| 换端口后忘了改 ufw | `ufw allow 2222` 漏了 |

## 5. 密码管理

```bash
# 生成 31 位随机密码
openssl rand -base64 24 | tr -d '\n' | head -c 31; echo

# 改当前用户密码
passwd
```

口令不写进笔记，用密码管理器保存。系统镜像默认口令（如 `ubuntu`）必须在首次登录后立刻改掉，否则等于对内网裸奔。

## 6. 常用速查

```bash
ssh ubuntu@192.168.1.100                              # 密码登录
ssh -i ~/.ssh/id_rsa_pi -p 2222 ubuntu@192.168.1.100  # 密钥 + 自定义端口
ip a show wlan0                                       # 在 Pi 上看 IP
```
