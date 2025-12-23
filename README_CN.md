# Pulse DNS

[English](README.md)

高性能 DNS 服务器，支持广告屏蔽、智能缓存和 Web 管理界面。

## 功能特性

### 多协议支持

- **UDP/TCP DNS** - 标准 DNS 协议，最大兼容性
- **DNS-over-HTTPS (DoH)** - 加密 DNS，支持 HTTP/1.1、HTTP/2 和 HTTP/3 (QUIC)
- **DNS-over-TLS (DoT)** - 隐私保护的加密 DNS
- **IPv4 & IPv6 双栈** - 客户端连接和上游服务器同时支持 IPv4 和 IPv6

### 智能缓存

- **Stale-While-Revalidate** - 立即返回缓存响应，后台异步刷新
- **灵活的 TTL 模式** - 固定值、倍数或遵循 DNS 原始 TTL
- **缓存持久化** - 重启不丢失缓存数据
- **可配置上限** - 设置最大缓存条目数（默认 100,000）

### 广告屏蔽

- **本地 Hosts 文件** - 加载多个本地屏蔽列表
- **远程更新** - 自动从 URL 获取并更新屏蔽列表
- **CNAME 链追踪** - 屏蔽隐藏在 CNAME 重定向后的广告（可配置深度）
- **屏蔽统计** - 统计屏蔽查询次数

### 上游管理

- **First-Wins 竞速** - 并发查询所有上游，使用最快的响应
- **多协议上游** - 混合使用 UDP、DoH、DoT 上游服务器
- **性能监控** - 追踪每个上游的响应时间和获胜率

### 自定义 DNS 记录

- **9 种记录类型** - A、AAAA、CNAME、TXT、MX、SRV、NS、PTR、CAA
- **动态管理** - 通过 REST API 添加/删除记录，无需重启
- **最高优先级** - 自定义记录优先于上游查询

### Web 管理界面

- **实时仪表板** - 监控查询数、缓存命中、屏蔽请求
- **缓存管理** - 浏览、搜索、清除缓存条目
- **屏蔽列表** - 查看所有被屏蔽的域名
- **记录管理** - 管理自定义 DNS 记录
- **国际化** - 中英文界面

## 性能

- **无 GC、无 STW** - 零垃圾回收停顿，确保持续低延迟响应
- 全协议高并发支持（UDP、TCP、DoH HTTP/1.1/2/3、DoT）
- 无锁高性能缓存查询
- 查询去重（singleflight），减少上游负载
- HTTP/3 (QUIC) 加速 DoH 查询

## 安装

### 下载

从 Releases 页面下载对应平台的版本：

**macOS**

| 架构 | DMG 安装包 | App 压缩包 | 归档文件 |
|------|-----------|-----------|---------|
| Apple Silicon (ARM64) | `pulse_dns-vX.X.X-aarch64-apple-darwin.dmg` | `.app.zip` | `.tar.gz` |
| Intel (x64) | `pulse_dns-vX.X.X-x86_64-apple-darwin.dmg` | `.app.zip` | `.tar.gz` |

**Linux**

| 架构 | AppImage | glibc 版本 | musl 静态版本 |
|------|----------|-----------|--------------|
| x64 | `pulse_dns-vX.X.X-x86_64.AppImage` | `-x86_64-unknown-linux-gnu.tar.gz` | `-x86_64-unknown-linux-musl.tar.gz` |
| ARM64 | `pulse_dns-vX.X.X-aarch64.AppImage` | `-aarch64-unknown-linux-gnu.tar.gz` | `-aarch64-unknown-linux-musl.tar.gz` |

**Windows**

| 架构 | 文件名 |
|------|--------|
| x64 | `pulse_dns-vX.X.X-x86_64-pc-windows-msvc.zip` |
| ARM64 | `pulse_dns-vX.X.X-aarch64-pc-windows-msvc.zip` |

### 目录结构

```
pulse_dns           # 主程序
config.toml         # 配置文件
static/             # Web 界面资源
hosts               # 本地屏蔽列表（可选）
custom_records.json # 自定义 DNS 记录（自动创建）
```

### 运行

```bash
# 使用默认配置（当前目录下的 config.toml）
./pulse_dns

# 指定配置文件
./pulse_dns -c /path/to/config.toml
```

## 配置说明

### 服务器配置

```toml
# UDP DNS 服务器
[udp_server]
enable = true
port = 53

# TCP DNS 服务器
[tcp_server]
enable = false
port = 53

# DoH 服务器（HTTPS）
[http_server]
enable = true
port = 443
cert = "cert.pem"
key = "key.pem"

# DoH 服务器（HTTP，用于反向代理）
[http_plain_server]
enable = false
port = 8053

# DoT 服务器
[tls_server]
enable = true
port = 853
cert = "cert.pem"
key = "key.pem"
```

### 上游服务器

```toml
# UDP 上游
[[upstream_server]]
enable = true
name = "223.5.5.5"
protocol = "udp"
port = 53

# DoT 上游
[[upstream_server]]
enable = true
name = "1.1.1.1"
protocol = "dot"
port = 853
sni = "cloudflare-dns.com"

# DoH 上游
[[upstream_server]]
enable = true
name = "cloudflare-doh"
protocol = "doh"
url = "https://1.1.1.1/dns-query"
http_version = 2  # 1、2 或 3

# 上游过滤选项
[upstream]
skip_blocked_response = false  # 跳过 0.0.0.0/127.0.0.1 响应
skip_empty_response = false    # 跳过空响应
```

### 缓存配置

```toml
[cache]
# Stale-while-revalidate 模式
stale_while_revalidate = true

# TTL 模式：fixed（固定）| multiplier（倍数）| dns_ttl（遵循原始）
ttl_mode = "fixed"
ttl_seconds = 86400        # 固定模式使用
ttl_multiplier = 2.0       # 倍数模式使用
min_ttl_seconds = 60       # 最小 TTL
max_ttl_seconds = 86400    # 最大 TTL

# 持久化
persist_on_shutdown = true
max_entries = 100000
```

### 广告屏蔽

```toml
[ad_block]
enable = true

# 本地 hosts 文件
hosts_files = ["hosts", "hosts.local"]

# 远程屏蔽列表 URL
update_urls = [
    "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts"
]
update_interval_seconds = 86400  # 24 小时

# CNAME 追踪
cname_blocking = true
cname_max_depth = 8
block_ttl_seconds = 60

# 统计
count_blocked_hits = true
```

### 管理界面

```toml
[admin]
enable = true
bind = ":8080"
username = "admin"
password = "your_password"
jwt_secret = "your_secret_key"
jwt_expiry_seconds = 86400
static_dir = "static"
records_file = "custom_records.json"

# 信任的代理（用于 X-Forwarded-For）
trusted_proxies = ["127.0.0.1", "192.168.0.0/16"]
# 或信任所有：trusted_proxies = ["*"]
```

### 统计配置

```toml
[stats]
count_total_queries = true
count_cache_hits = true
count_stale_hits = true
```

## 使用方法

启动服务器后，你可以：

1. **配置设备** 使用 DNS 服务器 IP 地址
2. **访问管理界面** `http://服务器IP:8080`
3. **测试 DNS 解析**：
   ```bash
   # 测试标准查询
   dig @服务器IP example.com

   # 测试广告屏蔽（应返回 0.0.0.0）
   dig @服务器IP ad.doubleclick.net
   ```
