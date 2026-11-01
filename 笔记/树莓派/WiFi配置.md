# 树莓派连接 Wi-Fi

> 适用：Raspberry Pi 4 + Ubuntu Server，无显示器时通过串口（115200）操作。无线网卡通常是 `wlan0`。

## 1. netplan 配置

```bash
ip link                      # 确认网卡名，通常是 wlan0
sudo nano /etc/netplan/99-wifi.yaml
```

```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: true
      optional: true
      access-points:
        "你的SSID":
          password: "你的密码"
```

三个易错点：

- **缩进全是空格**，不能用 Tab；`access-points` 下多一级缩进。
- SSID 和密码**必须加双引号**，含空格或特殊字符时尤其重要。
- 文件名必须是 `*.yaml` 且排序在系统默认配置之后（`50-cloud-init.yaml` 之后，用 `99-` 前缀即可）。

```bash
sudo chmod 600 /etc/netplan/99-wifi.yaml    # 含明文密码，netplan 会警告权限过宽
sudo netplan apply
```

## 2. 缺 wpa_supplicant 时

```bash
sudo apt update
sudo apt install -y wpasupplicant
sudo reboot
```

Ubuntu Server 精简镜像可能没带，报错形如 `wpa_supplicant: not found`。

## 3. 验证

```bash
ip a show wlan0
# 应有 inet 地址，状态 UP / LOWER_UP

ping -c 3 8.8.8.8
```

国内环境可能出现「连上了但没网」，先试设置监管域：

```bash
sudo iw reg set CN
```

## 4. 调试

```bash
sudo netplan --debug apply        # 看 YAML 解析和每一步执行细节
journalctl -u systemd-networkd -n 50
```

## 5. 坑

| 现象 | 原因 |
| --- | --- |
| `Error in network definition: unknown key` | YAML 缩进错或用了 Tab |
| 应用后 `wlan0` 无 inet | SSID/密码错，或 5G-only 热点 Pi 4 连不上（改用 2.4G 验证） |
| 重启后 Wi-Fi 丢失 | 配置文件权限过宽被忽略，或文件名排序被别的 yaml 覆盖 |
| `netplan apply` 卡住 | 有线网卡走 DHCP 无响应，加 `optional: true` |
| 两个 yaml 都配了 wlan0 | netplan 后者覆盖前者，重复定义会整段失效 |

## 6. 相关

- SSH 开启与加固：[SSH开启与加固.md](SSH开启与加固.md)
- 外网访问：IPv6 + 动态域名更新，见 [Nginx-PHP环境.md](Nginx-PHP环境.md) 末尾的定时上报
