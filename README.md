# mosdns — DNS 转发器 + Grafana 监控方案
Mosdns：https://github.com/IrineSistiana/mosdns
luci-app-mosdns：https://github.com/sbwml/luci-app-mosdns
icyleaf vector + loki 实现 mosdns 数据看板：https://icyleaf.com/2023/08/using-vector-transform-mosdns-logging-to-grafana-via-loki/#prometheus
mosdns_docker：https://github.com/Jasper-1024/mosdns_docker
cloudflare优选IP：https://github.com/XIU2/CloudflareSpeedTest

基于 [mosdns](https://github.com/IrineSistiana/mosdns) v5 的 DNS 分流部署方案。在Jasper-1024的Docker版及icyleaf的vector + loki 实现 mosdns 数据看板资料上进行的二创，实现在 OpenWrt 路由器上运行 mosdns，通过 Docker Compose 一键部署 Loki + Prometheus + Grafana 监控栈，实时查看 DNS 查询日志、指标监控和 Grafana 可视化面板。默认配置已测试DNS 300次TEST无泄漏。
项目采用监控面板Docker，Mosdns分布式部署，解决路由器容量空间受限导致Docker运行困难，环境混乱等问题，二创同时完成汉化及增加性能面板等。运行效果可看 Screenshot.jpeg。

代码助手：DeepseekV4Pro

## 架构

```text
OpenWrt 路由器 (192.168.11.1)            Docker 主机 (192.168.11.200)
┌──────────────────────────┐     ┌──────────────────────────────────┐
│  mosdns (UDP/TCP :5335)  │     │  vector (:9010)                  │
│  ├─ DNS 分流 + 缓存      │     │  ├─ 接收日志 → 解析 → 中文映射    │
│  ├─ Prometheus metrics   │────▶│  └─ 写入 events.json             │
│  │  (:8338)              │     │                                   │
│  └─ /var/log/mosdns.log  │     │  promtail                         │
│      │                    │     │  ├─ 读取 events.json              │
│      │ socat TCP ─────────┼────▶│  └─ push → Loki (:3100)          │
│      │                    │     │                                   │
│  mosdns-log (procd)      │     │  Loki (:3100)                     │
│  tail -f → socat TCP     │     │  └─ 日志存储 + 查询               │
└──────────────────────────┘     │                                   │
                                 │  Prometheus (:9090)               │
                                 │  └─ scrape mosdns :8338/metrics   │
                                 │                                   │
                                 │  Grafana (:3000)                  │
                                 │  ├─ Loki 数据源 → 日志面板         │
                                 │  └─ Prometheus 数据源 → 指标面板   │
                                 └──────────────────────────────────┘
```

## 运行环境

| 环境 | 要求 |
| ---- | ---- |
| **路由器** | OpenWrt，已安装 mosdns v5 + socat |
| **Docker 主机** | Docker Engine 24+ / Docker Compose v2 |
| **网络** | 路由器和 Docker 主机在同一局域网，互通 |

## 组件版本

| 组件 | 镜像/版本 |
| ---- | --------- |
| mosdns | v5 (OpenWrt 端，非 Docker) |
| Vector | `timberio/vector:latest-alpine` |
| Loki | `grafana/loki:3.4.3` |
| Promtail | `grafana/promtail:3.4.3` |
| Prometheus | `prom/prometheus:latest` |
| Grafana | `grafana/grafana:latest` |

## 目录结构

```text
mosdns/
├── README.md
├── dashboard/
│   ├── .env                          # 统一环境变量
│   ├── docker-compose.yaml           # 监控栈编排
│   ├── vector/config.yml             # Vector 日志解析 + 中文映射
│   ├── loki/loki-config.yaml         # Loki 存储配置
│   ├── promtail/promtail-config.yaml # Promtail 采集配置
│   ├── prometheus/prometheus.yml     # Prometheus scrape 配置
│   ├── panel/panel.json              # Grafana 面板 (21 个面板)
│   └── mosdns/
│       ├── config/config_custom.yaml # mosdns DNS 分流核心配置
│       └── mosdns-log               # OpenWrt procd 日志转发脚本
```

## 安装步骤

### 1. 路由器端 — 安装 mosdns

在 OpenWrt 路由器上安装 mosdns v5：

```bash
# 下载 mosdns v5 二进制 (替换为实际架构, 如 linux_amd64, linux_arm64)
wget https://github.com/IrineSistiana/mosdns/releases/latest/download/mosdns-linux-amd64.zip
unzip mosdns-linux-amd64.zip -d /usr/bin/
chmod +x /usr/bin/mosdns
```
建议安装luci-app-mosdns：https://github.com/sbwml/luci-app-mosdns，配合其“自定义配置”功能及“GeoData导出”功能使用。这样可以跳过上传规则文件和数据文件。
GeoData导出GeoSite标签：cn,gfw,apple,category-ads-all,geolocation-!cn,disney,hulu,netflix
GeoData导出GeoIP标签：cn,private

更新广告规则、GeoIP & GeoSite 数据库的时间根据自己喜欢来定。

### 2. 路由器端 — 部署配置文件

```bash
# 创建目录结构
mkdir -p /etc/mosdns/rule /var/mosdns

核心配置加入大量的注释，方便大家对照调整，其中需修改：
forward_local中的DNS服务器地址，根据自己喜欢来，默认配置了223.5.5.5及119.29.29.29。（也可配置运营商DNS，我测试无泄露，这样能得到最快的CDN）
has_resp_sequence中的black_hole里的IP地址，这是使用Cloudflare优选IP找到的适合自己最快的地址，可以多填。每个IP地址用空格分隔。
然后改名为：config_custom.yaml

# 上传核心配置
scp dashboard/mosdns/config/config_custom.yaml root@192.168.11.1:/etc/mosdns/


（第一步使用GeoData自动导出的话，可以省略以下两步）
# 上传规则文件 (whitelist.txt, blocklist.txt 等)
scp dashboard/mosdns/rule/*.txt root@192.168.11.1:/etc/mosdns/rule/

# 上传 geosite/geoip 数据文件 (需先用 v2ray-rules-dat 生成)
scp geosite_cn.txt geoip_cn.txt geosite_apple.txt ... root@192.168.11.1:/var/mosdns/
```

### 3. 路由器端 — 配置日志转发
先安装socat，opkg update && opkg install socat
```bash
# 上传 init.d 脚本
scp dashboard/mosdns/mosdns-log root@192.168.11.1:/etc/init.d/

# 修改脚本中的 DOCKER_HOST 为实际 Docker 主机 IP (默认 192.168.11.200)
vi /etc/init.d/mosdns-log

# 启用并启动
chmod +x /etc/init.d/mosdns-log
/etc/init.d/mosdns-log enable
/etc/init.d/mosdns-log start
```

### 4. 路由器端 — 启动 mosdns

```bash
# 前台测试运行
mosdns run -c /etc/mosdns/config_custom.yaml -d /etc/mosdns
如果没有出现错误，则说明配置文件格式正确。可以CTRL+C终止运行后，使用luci-app-mosdns，选择自定义配置来运行。

# 验证: metrics 端点
curl http://192.168.11.1:8338/metrics

# 验证: DNS 查询
nslookup baidu.com 192.168.11.1#5335
```

### 5. Docker 主机 — 部署监控栈

```bash
cd dashboard

# 检查并修改环境变量 (.env)
# 确保 ROUTER_IP、DOCKER_HOST_IP 与你的网络一致

# 启动所有容器
docker compose up -d

# 验证各组件
docker compose ps
curl http://localhost:3100/ready  # Loki
curl http://localhost:9090/metrics # Prometheus
```

### 6. 导入 Grafana 面板

1. 打开浏览器访问 `http://192.168.11.200:3000`
2. 左侧菜单 → Dashboards → New → Import
3. 上传 `dashboard/panel/panel.json`

## mosdns DNS 分流逻辑

`config_custom.yaml` 中 main_sequence 按优先级执行 14 步分流：

| 步骤 | 规则 | 上游 |
| ---- | ---- | ---- |
| 1 | Prometheus 指标采集 | — |
| 2 | 黑名单/PTR/HTTPS | 直接拒绝 |
| 3 | 广告域名 (默认注释) | 拒绝 |
| 4 | 本地 hosts 文件 | hosts 文件 |
| 5 | redirect 重定向 | redirect 规则 |
| 6 | DNS 缓存 | lazy_cache (300-86400s) |
| 7 | Apple 域名 | 国内 DNS + 114 DNS 备用 |
| 8 | DDNS 动态域名 | 国内 DNS (TTL 5s) |
| 9 | 自定义白名单 | 国内 DNS |
| 10 | 流媒体域名 | Google DNS (8.8.8.8) |
| 11 | 灰名单 (防污染) | Google + Cloudflare |
| 12 | 国内域名 (geosite_cn) | 国内 DNS 池 |
| 13 | 非国内域名 | Google + Cloudflare |
| 14 | 兜底竞速 | Google vs Cloudflare 并行 |

## 配置文件说明

### .env — 统一环境变量

所有用户可配置项集中在 `.env`：

- `ROUTER_IP` / `DOCKER_HOST_IP` — 网络地址
- 各组件端口号
- Grafana 数据源 UID (需与 panel.json 一致)
- Prometheus scrape 目标

### config_custom.yaml — mosdns 核心配置

包含 52 个插件定义 + 主路由序列，详见文件内注释。

## 许可

GNU General Public License v3.0

---

二创：Jerrell
