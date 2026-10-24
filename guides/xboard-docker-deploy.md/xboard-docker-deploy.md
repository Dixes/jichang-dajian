# Xboard 面板 Docker 部署教程

Xboard 是目前机场运营里常用的面板之一，负责用户、套餐、订单、节点和订阅的统一管理。面板本身不转发流量，它是控制面：你在后台建节点、定套餐，用户通过它注册、购买并获取订阅。本文用 Docker 在一台境外 VPS 上把 Xboard 跑起来，并完成初始化、反代和基础安全设置，后续对接 V2bX 节点端才有地方可连。

## 准备工作

准备一台 Debian 12 或 Ubuntu 22.04 的 VPS，建议 2 核 2GB 内存起步，因为面板、数据库和 Redis 会同时常驻；一个已解析到服务器的域名，例如 panel.example.com；以及 SSH 登录权限。先更新系统并安装 Docker：

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
docker compose version
```

最后一条能输出版本号，说明 Compose 插件可用，这是后面一条命令拉起全部服务的关键。开放防火墙的 80、443 和 SSH 端口，面板的内部端口不要直接暴露到公网，后面会用反代转发，这样能减少扫描和爆破面。

## 拉取项目并配置

创建部署目录并拉取 Xboard 的 Docker 部署文件，不同发行版的文件名略有差异，以官方仓库的 docker-compose 部署说明为准：

```bash
mkdir -p /opt/xboard && cd /opt/xboard
git clone https://github.com/cedar2025/Xboard.git .
cp .env.example .env
nano .env
```

`.env` 里重点改三类参数：数据库密码、Redis 密码和站点地址。密码用随机长字符串，因为数据库一旦被弱口令打穿，用户和订单数据会全部泄露；站点地址必须写最终对外访问的域名，否则订阅链接和支付回调会生成错误地址。

## 启动服务

```bash
docker compose up -d
docker compose ps
docker compose logs -f --tail=100
```

`up -d` 会拉起 Web、数据库、Redis 等容器。`ps` 里所有服务都应是 running 或 healthy，日志里反复重启通常是 `.env` 配置错误或端口被占用，先看日志第一条报错，不要盲目重装。首次启动后按官方提示执行初始化命令创建管理员账号，命令执行完妥善保存账号密码，不要写进公开仓库或截图发群。

## 配置反代与 HTTPS

面板建议套一层 Caddy 或 Nginx 反代并启用 HTTPS，细节在证书专篇展开，这里先保证能用域名访问：

```nginx
server {
    listen 80;
    server_name panel.example.com;
    location / {
        proxy_pass http://127.0.0.1:7001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

端口 7001 只是示例，要与 compose 里映射的宿主机端口一致。`X-Real-IP` 必须传递，否则后台看到的全是反代的本机 IP，用户登录风控和审计都会失真。签发证书后把 80 跳转到 443，并确认浏览器地址栏没有证书警告。


## 初始化后台设置

登录后台后按顺序做四件事：第一，改站点名称、站点地址和注册开关，内测期建议关闭公开注册，避免还没上线就被机器人灌号；第二，配置邮件 SMTP，发信用于找回密码和订单通知，发送前先给自己发一封测试信；第三，设置时区和货币，避免套餐周期和订单时间错位；第四，在系统设置里确认订阅域名和面板域名分离的规划，后期换面板服务器时只改解析就能迁移。

## 验证是否部署成功

依次验证：用域名打开首页能正常加载样式；管理员能登录且刷新不掉线；新建一个测试用户，后台能查到；容器重启后数据仍在：

```bash
docker compose restart
docker compose ps
```

数据还在说明挂载卷生效，如果重启后管理员消失，多半是把数据写进了容器临时层，检查 compose 的 volumes 配置。

## 常见坑

一是 502 Bad Gateway，多数是反代端口与容器映射端口对不上，用 `curl 127.0.0.1:端口` 先在本机验证再查反代。二是安装后页面空白或接口 500，通常是数据库连接信息改了却没同步 `.env`，改完必须重新 `up -d` 才会生效。三是时间不对导致订阅和订单异常，给宿主机开启 NTP 同步。四是把数据库端口映射到公网，这是最危险的误操作，数据库只应让容器内网访问。

## 不想自己搭？

自己搭机场，前期部署只是开始，后面还有域名、证书、节点维护、线路排查和备份迁移持续占用时间。如果你只是想稳定使用、不想长期运维，可以考虑漫画机场的现成订阅作为替代：标准套餐 ¥10/月起、家宽 ¥30/月起，提供 100GB/月流量、可同时使用 3 台设备，官网：https://vps.manhuajichang.fun 。
