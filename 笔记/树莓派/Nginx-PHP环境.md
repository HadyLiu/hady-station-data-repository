# 树莓派：Nginx + PHP-FPM（对外 88 端口）

> 适用：Raspberry Pi（Ubuntu Server）。用途：让 Pi 同时提供静态页和 JSON 接口，配合 IPv6 + 动态域名从外网访问。
> 外网示例：`http://<你的域名>:88/`

## 1. 架构

```
浏览器 ──:88──► Nginx ──┬─► 静态文件 (html/css/js)
                        └─► PHP-FPM ─► *.php（动态页 / JSON 接口）
```

- **Nginx**：只做反向代理和静态资源，`.php` 请求转给 PHP-FPM（unix socket）。
- **PHP-FPM**：常驻进程池，执行 PHP 逻辑。
- 浏览器端仍然需要 HTML/CSS/JS；PHP 只决定服务端返回什么。

网页目录：`/var/www/pi-php/`

## 2. 控制脚本 `pi-php-ctl.sh`

| 命令 | 作用 |
| --- | --- |
| `./pi-php-ctl.sh setup` | 装 nginx + php-fpm、部署页面、写站点配置、**设开机自启** |
| `./pi-php-ctl.sh start` | 启动服务并启用 :88 站点 |
| `./pi-php-ctl.sh stop` | 摘掉 :88 站点（服务本身可仍在跑） |
| `./pi-php-ctl.sh restart` | 先 stop 再 start |
| `./pi-php-ctl.sh status` | 查 nginx / php-fpm 是否 active、是否 enabled |
| `./pi-php-ctl.sh enable` | 只设开机自启 |
| `./pi-php-ctl.sh disable` | 取消开机自启 |

环境变量：

- `PORT`：默认 `88`
- `WEB_ROOT`：默认 `/var/www/pi-php`

```bash
PORT=8080 ./pi-php-ctl.sh setup
```

## 3. 首次部署

```bash
cp -r pi-php ~/pi-php && cd ~/pi-php
chmod +x pi-php-ctl.sh
./pi-php-ctl.sh setup
./pi-php-ctl.sh status
```

访问：

- 首页 `http://<你的域名>:88/`
- 接口 `http://<你的域名>:88/api.php`

改页面直接编辑 `/var/www/pi-php/index.php`，保存即生效；**只有改了 nginx 配置才需要 reload**。

## 4. 开机自启做了什么

`setup` / `enable` 大致等价于：

```bash
sudo systemctl enable --now nginx
sudo systemctl enable --now php*-fpm
```

并保证 `/etc/nginx/sites-enabled/pi-php88` 这个软链接存在。检查：

```bash
systemctl is-enabled nginx
systemctl is-enabled php8.3-fpm     # 版本号以本机为准
ss -tlnp | grep 88
```

## 5. 外网访问（IPv6 + 动态域名）

家庭宽带 IPv4 多为 NAT，用 IPv6 更省事（前提：光猫/路由放行入站，部分光猫会拦 IPv6 入站——那就得走 Cloudflare Tunnel 之类）。

动态域名更新脚本 + 定时器：

```bash
chmod +x dynv6-update-ipv6.sh
./dynv6-update-ipv6.sh              # 先手动跑通

crontab -e
# 每 5 分钟上报一次
*/5 * * * * /home/ubuntu/dynv6-update-ipv6.sh >>/tmp/dynv6.log 2>&1
```

IPv6 前缀会随运营商重播变化，间隔别设太长（≤15 分钟），日志记得清理。

## 6. 相关脚本对照

| 文件 | 用途 |
| --- | --- |
| `dynv6-update-ipv6.sh` | 上报 IPv6 到动态域名 |
| `pi-php-ctl.sh` | **当前推荐**：Nginx + PHP-FPM 管理 |
| `pi-www-ctl.sh` | 早期纯静态 Nginx 开关（可弃用） |
| `pi-flask/` | Flask 方案，已改用 PHP 可忽略 |

## 7. 坑

| 现象 | 原因 |
| --- | --- |
| 打开是 Nginx 欢迎页 | 站点没启用（`sites-enabled` 无软链接），或 80 端口的 default 抢了请求 |
| 下载 `.php` 而不是执行 | 缺 `location ~ \.php$` 或 `fastcgi_pass` 指向错 |
| `502 Bad Gateway` | PHP-FPM 没起 / socket 路径不匹配（`/run/php/php8.3-fpm.sock`） |
| 外网不通、内网正常 | 光猫拦 IPv6 入站，或 `ufw` 未放行：`sudo ufw allow 88/tcp` |
| 改完配置不生效 | 只改了 `/etc/nginx/sites-available/` 没建 `sites-enabled` 软链接 |
