# VoHive

[![License: PolyForm Noncommercial 1.0.0](https://img.shields.io/badge/License-PolyForm--Noncommercial--1.0.0-blue.svg)](https://polyformproject.org/licenses/noncommercial/1.0.0)
[![Go](https://img.shields.io/badge/Go-1.26%2B-00ADD8?logo=go)](go.mod)
[![Vue 3](https://img.shields.io/badge/Vue-3-42b883?logo=vue.js)](web/package.json)

> 面向高通 4G/LTE/5G 模组（Quectel EC20/EC25/EC21/EG25/EM20 等）的综合管理与代理服务平台。

VoHive 把模组热插拔管理、SOCKS5/HTTP 代理编排、短信收发、VoWiFi/IMS 通话、eSIM 全生命周期管理整合到一个服务里,并提供一套现代化的响应式 Web 管理后台。目标是替代生产环境里那种东拼西凑的单脚本方案,给多模组场景一套开箱即用、可长期运维的一体化系统。

## 核心特性

| 模块 | 说明 |
| --- | --- |
| 多模组并发管理 | USB 热插拔自动发现(ttyUSB 等)、多设备实时状态监控 |
| 轻量级代理引擎 | 内建 SOCKS5 / HTTP 代理内核,支持多实例并发;基于 `SO_BINDTODEVICE` 按设备网卡严格绑定出站流量 |
| 通信与短信中心 | 统一界面/API 处理 AT 短信收发、会话与联系人管理、USSD 交互,短信落库可查 |
| VoWiFi / IMS | 完整 ePDG/IPSec、IMS 注册、SMS over IP,内置语音网关,可通过标准 SIP 对接 Linphone 等软电话,在零蜂窝信号下用 Wi-Fi 打电话收短信 |
| eSIM 管理 | 通过 AT 指令通道直接管理 eSIM 芯片,支持 Profile 下载、启用/停用、重命名、删除 |
| 全渠道通知 | 重要短信及系统告警可推送至 Telegram、Email、PushPlus、Bark、飞书(Lark/Feishu)、QQ 等 |
| 多架构构建 | 原生支持 amd64 / arm64 / arm7 跨平台编译,路由器到边缘节点均可部署 |

## 典型应用场景

- **私有 IP 代理池**:单主机挂载多张物理 SIM 卡或多张 eSIM,每张网卡对应独立的 SOCKS5/HTTP 实例,组建自己的移动网络代理池。
- **统一接码/验证码中心**:Web 界面或 API 并行收发多卡短信,并通过 Webhook/Bot 实时推送到个人终端。
- **VoWiFi 零信号通信**:地下室、弱覆盖场景下,借助宽带网络隧道建立 IMS 连接,保证业务不掉线。
- **无人值守自动化运维**:配合内置 API 与通知机制,实现自动拨测、自动 USSD 查费、流量重载等。

## 架构与技术栈

- **Backend**:Go 1.26+(Gin、GORM、Viper、sipgo、euicc-go)
- **Frontend**:Vue 3 + Vite + TailwindCSS + Element Plus
- **Database**:SQLite(`vohive.db`)
- **CI/CD**:GitHub Actions 自动化多架构 Docker 镜像构建与发布

关键目录:

```
cmd/vohive          主服务入口
internal/api         REST API 控制平面
internal/device       设备发现、拨号与状态生命周期
internal/proxy        代理实例管理与流量统计监控
internal/vowifi        ePDG/IMS 协议栈与软电话网关
internal/notify        统一告警与消息推送机制
web                  管理后台前端源码
```

## 快速开始

### 1) 源码本地构建

依赖:Go 1.26+,Node.js 18+。

```bash
# 构建前端静态文件
cd web
npm ci
npm run build
cd ..

# 拷贝静态文件用于嵌入
mkdir -p internal/web/dist
cp -r web/dist/* internal/web/dist/

# 编译后端二进制(禁用 CGO 实现完全静态编译)
CGO_ENABLED=0 go build -trimpath -buildvcs=false -tags "with_utls nomsgpack" -ldflags "-s -w" -o vohive ./cmd/vohive
```

也可以直接用 `Makefile`:

```bash
make build-amd64   # 或 build-arm64 / build-armv7 / build-all
```

### 2) 运行服务

```bash
# 使用默认配置运行(读取 config/config.yaml)
./vohive

# 指定配置文件运行
./vohive -c /path/to/custom_config.yaml
```

- 默认管理后台:`http://127.0.0.1:7575`
- 默认账密:`admin / admin`(请在首次运行后立即在配置中修改)

### 3) 容器化部署(Docker)

代理流量精确路由和底层硬件串口访问依赖宿主网络与设备直通,Docker 部署强烈建议使用 `host` 网络及特权模式。

```bash
docker run -d \
  --name vohive \
  --network host \
  --privileged \
  -v /dev:/dev \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/data:/app/data \
  vohive:latest
```

也可以使用仓库内的 `docker-compose.yml` / `docker-compose.hub.yml` 部署。

## 配置文件示例

VoHive 使用 YAML 配置,首次运行会自动生成 `config/config.yaml`。所有配置项均可通过环境变量覆盖,前缀为 `PROXY_`(例如 `PROXY_WEB_USERNAME=admin`)。

```yaml
server:
  port: ":7575"
  debug: false

web:
  username: "admin"
  password: "admin"

proxy:
  instances:
    - id: "proxy-socks-1"
      name: "SOCKS5-Dev1"
      device_id: "ec20_1"
      enabled: true
      mode: "socks5"
      listen_addr: "0.0.0.0"
      listen_port: 10800
      auth_enabled: false

notifications:
  telegram:
    enabled: true
    bot_token: "xxx"
    chat_id: 123456
  email:
    enabled: true
    smtp_server: "smtp.example.com"
  # ... 等等
```

## API 接入

Web 管理平台的所有操作均可通过 REST API 实现,调用需携带 Header:`Authorization: Bearer <token>`。

服务启动后内置完整接口文档,无需额外查阅源码:

- `GET /api/docs` — 交互式 API 文档
- `GET /api/openapi.json` / `GET /api/openapi.yaml` — OpenAPI 规范,可导入 Postman / Apifox 等工具

常用端点举例:

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/proxy-instances/overview` | 代理实例总览与实时流量统计 |
| GET | `/api/devices` | 已连接模组列表及运行状态 |
| GET | `/api/devices/:device_id/esim/profiles` | 查询设备的 eSIM Profile 列表 |
| POST | `/api/sms/send` | 发送短信(自动选择 AT 或 VoWiFi 通道) |
| POST | `/api/devices/:device_id/vowifi/actions/reconnect` | 重连 VoWiFi |

## License

本项目基于 [PolyForm Noncommercial License 1.0.0](LICENSE) 开源,**仅限非商业用途**:可自由查看、使用、修改、分发源码用于个人学习、研究、测试等非商业场景;**禁止任何形式的商业使用**(包括但不限于销售、提供付费服务、用于盈利性产品或业务)。如需商业授权,请联系作者另行协商。
