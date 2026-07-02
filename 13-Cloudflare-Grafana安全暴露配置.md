# Cloudflare + Grafana 安全暴露配置指南

> 通过 Cloudflare 代理将 Grafana 以 `https://grafana.spock.cloud` 暴露，源 IP 隐藏，安全组仅放行 Cloudflare
>
> 撰写人：孟希東

---

## 架构

```
用户浏览器
  ↓ https://grafana.spock.cloud
Cloudflare Edge（SSL 终止 + CDN + WAF）
  ↓ HTTP:80（Flexible 模式）
VM 安全组（仅允许 Cloudflare IP 入站）
  ↓
Grafana（0.0.0.0:80）
```

| 组件 | 说明 |
|------|------|
| Cloudflare DNS | A 记录 `grafana` → 源 IP，开启 Proxied（橙色云朵） |
| Cloudflare SSL | Flexible 模式，用户侧 HTTPS，回源 HTTP |
| VM 安全组 | 端口 80 仅对 Cloudflare IP 段开放 |
| Grafana | 直接监听 0.0.0.0:80，无需 Nginx |

---

## Step 1 · Grafana 配置监听 80 端口

编辑配置文件：

```bash
sudo nano /etc/grafana/grafana.ini
```

修改 `[server]` 部分：

```ini
[server]
http_port = 80
http_addr = 0.0.0.0
domain = grafana.spock.cloud
root_url = https://grafana.spock.cloud/
```

> `root_url` 写 https 是因为用户实际通过 Cloudflare SSL 访问，Grafana 生成的链接（OAuth 回调、重定向等）需要匹配。

### 授权低端口绑定

Linux 默认不允许非 root 进程监听 1024 以下端口：

```bash
# 方式一：setcap 授权（裸装 Grafana）
sudo setcap 'cap_net_bind_service=+ep' /usr/sbin/grafana-server

# 方式二：Docker 部署（容器内 3000 映射到主机 80）
docker run -d --name grafana -p 80:3000 grafana/grafana
```

重启：

```bash
sudo systemctl restart grafana-server

# 验证
curl -I http://localhost:80
# 应返回 200 / 302
```

---

## Step 2 · Cloudflare DNS 配置

登录 Cloudflare Dashboard → 选择 `spock.cloud` → **DNS** → **Records** → **Add Record**：

| 字段 | 值 |
|------|-----|
| Type | **A** |
| Name | **grafana** |
| IPv4 address | **89.167.96.196** |
| Proxy status | **Proxied**（橙色云朵 ☁️） |
| TTL | Auto |

> 开启 Proxied 后，`dig grafana.spock.cloud` 返回的是 Cloudflare 的 IP，源 IP 被隐藏。

---

## Step 3 · Cloudflare SSL 配置

### 3.1 SSL 模式

**SSL/TLS** → **Overview** → 选择 **Flexible**

| 模式 | 链路加密 | 源站要求 |
|------|----------|----------|
| **Flexible** | 用户↔CF 加密，CF↔源站明文 | 源站无需证书 ✅ |
| Full | 全程加密 | 源站需配证书 |
| Full (Strict) | 全程加密 + 验证源站证书 | 源站需有效证书 |

> Flexible 最简单：源站只监听 HTTP:80，用户侧通过 Cloudflare 自动获得 HTTPS。安全组已限制只有 Cloudflare 能访问源站，中间链路安全可控。

### 3.2 强制 HTTPS

**SSL/TLS** → **Edge Certificates** → 开启：

- **Always Use HTTPS** → On
- **Automatic HTTPS Rewrites** → On

---

## Step 4 · VM 安全组配置

在云厂商控制台（或 iptables/ufw）配置入站规则，**端口 80 仅对 Cloudflare IP 开放**：

### Cloudflare IPv4 地址段（2026 年）

```
173.245.48.0/20
103.21.244.0/22
103.22.200.0/22
103.31.4.0/22
141.101.64.0/18
108.162.192.0/18
190.93.240.0/20
188.114.96.0/20
197.234.240.0/22
198.41.128.0/17
162.158.0.0/15
104.16.0.0/13
104.24.0.0/14
172.64.0.0/13
131.0.72.0/22
```

### 安全组规则模板

| 方向 | 协议 | 端口 | 源 | 动作 |
|------|------|------|-----|------|
| 入站 | TCP | 80 | 173.245.48.0/20 | 允许 |
| 入站 | TCP | 80 | 103.21.244.0/22 | 允许 |
| 入站 | TCP | 80 | 103.22.200.0/22 | 允许 |
| 入站 | TCP | 80 | 103.31.4.0/22 | 允许 |
| 入站 | TCP | 80 | 141.101.64.0/18 | 允许 |
| 入站 | TCP | 80 | 108.162.192.0/18 | 允许 |
| 入站 | TCP | 80 | 190.93.240.0/20 | 允许 |
| 入站 | TCP | 80 | 188.114.96.0/20 | 允许 |
| 入站 | TCP | 80 | 197.234.240.0/22 | 允许 |
| 入站 | TCP | 80 | 198.41.128.0/17 | 允许 |
| 入站 | TCP | 80 | 162.158.0.0/15 | 允许 |
| 入站 | TCP | 80 | 104.16.0.0/13 | 允许 |
| 入站 | TCP | 80 | 104.24.0.0/14 | 允许 |
| 入站 | TCP | 80 | 172.64.0.0/13 | 允许 |
| 入站 | TCP | 80 | 131.0.72.0/22 | 允许 |
| 入站 | TCP | 22 | 你的管理 IP | 允许 |
| 入站 | 全部 | 全部 | 0.0.0.0/0 | **拒绝** |

### 如果用 UFW

```bash
# 重置规则
sudo ufw reset

# 允许 SSH
sudo ufw allow from <你的IP>/32 to any port 22

# 允许 Cloudflare 访问 80
for ip in 173.245.48.0/20 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 \
  141.101.64.0/18 108.162.192.0/18 190.93.240.0/20 188.114.96.0/20 \
  197.234.240.0/22 198.41.128.0/17 162.158.0.0/15 104.16.0.0/13 \
  104.24.0.0/14 172.64.0.0/13 131.0.72.0/22; do
  sudo ufw allow from $ip to any port 80
done

# 默认拒绝其他入站
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
sudo ufw status numbered
```

> Cloudflare IP 列表官方地址：https://www.cloudflare.com/ips-v4 ，建议定期核对更新。

---

## Step 5 · 验证

```bash
# 1. DNS 解析 — 应返回 Cloudflare IP，不是源 IP
dig grafana.spock.cloud +short
# 104.21.x.x / 172.67.x.x（Cloudflare IP）

# 2. HTTPS 访问 — 正常
curl -I https://grafana.spock.cloud
# HTTP/2 200

# 3. HTTP 自动跳转 HTTPS
curl -I http://grafana.spock.cloud
# HTTP/1.1 301 → https://grafana.spock.cloud/

# 4. 直连源 IP — 应被拒绝
curl --connect-timeout 5 http://89.167.96.196
# 超时或拒绝
```

浏览器打开 **https://grafana.spock.cloud** 即可访问。

---

## 安全加固（可选）

### Cloudflare WAF

**Security** → **WAF** → 添加规则，例如限制特定国家或 IP 访问。

### Cloudflare Access（零信任登录门）

**Zero Trust** → **Access** → **Applications** → Add → Self-hosted：

| 字段 | 值 |
|------|-----|
| Application domain | grafana.spock.cloud |
| Session duration | 24 hours |
| Policy | 允许指定邮箱域 / GitHub 账号 / OTP |

这样在 Grafana 登录页之前还有一层 Cloudflare 的身份验证。

### Grafana 自身安全

```ini
# /etc/grafana/grafana.ini

[security]
admin_password = YourStrongPassword@2026
disable_gravatar = true

[users]
allow_sign_up = false

[auth.anonymous]
enabled = false
```
