# NetPortal 内网穿透系统

> 一个支持**端口映射发布**与**虚拟组网互联**双形态、可私有化部署的内网穿透平台。
> 管理面基于 webman（PHP），数据面采用 C/C++ 高性能实现。

[![License](https://img.shields.io/badge/License-Commercial-blue)](#商业授权)
[![Version](https://img.shields.io/badge/Version-1.0.0-green)](#changelog)
[![Status](https://img.shields.io/badge/Status-GA-blue)](#路线图)

[简体中文](README.md) | [English](README.en.md)

---

## 目录

- [项目介绍](#项目介绍)
- [核心特性](#核心特性)
- [系统架构](#系统架构)
- [技术栈](#技术栈)
- [安装与使用](#安装与使用)
- [快速示例](#快速示例)
- [性能目标](#性能目标)
- [路线图](#路线图)
- [文档](#文档)
- [商业授权](#商业授权)
- [参与贡献](#参与贡献)
- [法律声明与免责条款](#法律声明与免责条款)

---

## 项目介绍

NetPortal 用于解决内网设备与服务因 NAT / 防火墙限制而无法被外部访问的问题，提供两种互补能力：

| 形态 | 典型场景 |
| --- | --- |
| **端口映射**：将内网服务发布到公网服务器端口 | 远程访问 NAS、监控、ERP、自建网站；对外提供 API 服务 |
| **虚拟组网（Overlay VPN）**：多台设备加入同一虚拟局域网 | 异地办公互联、连锁门店组网、工业设备远程运维 |

区别于 SaaS 穿透服务，本项目面向 **私有化部署售卖**：数据全程经由客户自己的服务器，控制台、授权、监控完全自主可控。

## 核心特性

### 端口映射
- TCP / UDP 服务发布，HTTP(S) 域名反代（vhost）
- 全链路 TLS 加密隧道，可选端到端私密穿透（访客模式）
- KCP/QUIC 弱网加速、流量压缩、P2P 打洞直连
- 按规则 / 节点两级带宽与连接数限速

### 虚拟组网
- 多虚拟网络（VN）逻辑隔离，虚拟 IP 自动分配
- TUN / wintun 虚拟网卡，跨 Linux / Windows / macOS / ARM
- NAT 打洞优先 P2P 直连，失败自动回落服务器中继
- 子网路由：一个节点可代理其整个物理子网供其他成员访问

### 管理控制台（webman）
- 多租户 RBAC、节点生命周期管理、配置热下发
- 实时流量监控、审计日志、OpenAPI 对接第三方系统
- License 授权管理（机器指纹绑定、离线激活）

> 注：当前版本管理后端采用 **JSON 文件存储**（`admin/data/*.json` + np-server `data/*.json`），开箱即用、零外部依赖；MySQL/Redis 为后续生产化规划，接口层已按可替换设计。

## 系统架构

```
                 ┌────────────────────────────────────────────┐
                 │              管理面 (webman)                │
  浏览器 ───────▶│  Console(Vue3) + Admin API + License 校验    │
  第三方系统 ────▶│  OpenAPI                                    │
                 └───────┬─────────────────┬──────────────────┘
                         │ MySQL           │ Redis
                         ▼                 ▼
┌────────────────┐  控制/配置同步   ┌──────────────────────┐
│ np-client      │◀═══TLS 信道═══▶ │ np-server (公网)      │
│ (内网主机/设备) │                 │  - 隧道接入网关        │◀── 外网访客
│  - 服务映射     │═══数据信道═══▶  │  - vhost 反代          │    (HTTP/TCP/UDP)
│  - 虚拟网卡     │  TCP/UDP/KCP    │  - P2P 信令 & 中继      │◀── VN 成员
└───────┬────────┘                 └──────────┬───────────┘
        │ TUN/wintun                          │ STUN
        ▼                                     ▼
   内网服务/局域网                        P2P 直连链路
```

| 组件 | 语言 | 说明 |
| --- | --- | --- |
| netportal-admin | PHP (webman) | 管理 API、控制台后端、租户与授权管理 |
| np-server | C/C++ | 公网隧道网关：转发、vhost 反代、P2P 信令与中继 |
| np-client | C/C++ | 内网客户端：服务映射、虚拟网卡、子网路由 |
| np-console | Vue3 + TS | Web 管理控制台前端 |

## 技术栈

- **数据面**：C++17 · epoll/IOCP（预留 io_uring）· OpenSSL/mbedTLS · Protobuf(nanopb) · wintun/TUN
- **管理面**：webman (workerman) · MySQL 8 · Redis 7
- **前端**：Vue 3 · TypeScript · ECharts
- **构建**：CMake（交叉编译支持嵌入式 ARM）

## 安装与使用

### 从源码构建

```bash
# 依赖：cmake ≥ 3.16, g++ (C++17), OpenSSL development headers
# Debian/Ubuntu: apt install cmake g++ libssl-dev
# CentOS/RHEL:   dnf install cmake gcc-c++ openssl-devel

cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)

# 二进制输出
ls build/np-server build/np-client
```

### 跨平台打包（Windows / macOS / Linux / Android）

一键交叉编译并打包多平台发布包：

```bash
# 本机平台（Linux x64，Ubuntu/CentOS 通用）
./build_release.sh --target linux-x64

# 全部可用平台（自动跳过未安装工具链的）
./build_release.sh --all

# 指定平台：linux-x64 / windows-x64 / android-arm64 / macos-x64
./build_release.sh --target windows-x64
```

各平台要求：

| 平台 | 工具链 | 说明 |
|------|--------|------|
| linux-x64 | g++ + cmake | 原生编译；二进制兼容 Ubuntu / Debian / CentOS（同为 glibc） |
| windows-x64 | mingw-w64 + OpenSSL(mingw) | `apt install mingw-w64`；OpenSSL 用 `./build_release.sh --openssl-windows` 自动编译 |
| android-arm64 | Android NDK + OpenSSL(android) | 设置 `ANDROID_NDK_HOME` 后使用 |
| macos | Apple clang（原生）或 osxcross | 需在 macOS 原生构建，或安装 osxcross（含 macOS SDK） |

产出物位于 `release/` 目录：`netportal-<版本>-<平台>.tar.gz`（或 `.zip`），
内含 `bin/`（np-server / np-client）、`config/`（server.ini / client.ini 示例）、`docs/`。

### 服务端部署（公网机器）

```bash
# 创建配置文件
cat > server.ini <<'EOF'
[common]
bind_ip = 0.0.0.0
control_port = 7000
data_port = 7200

[license]
key = <your-license-key>
EOF

# 启动
./build/np-server -c server.ini
```

### 客户端部署（内网机器）

```bash
# 创建配置文件
cat > client.ini <<'EOF'
[common]
server_addr = your-server-ip
server_port = 7000
data_port = 7200
token = <node-token>

[mappings]
ssh = 127.0.0.1:22
web = 127.0.0.1:8080
EOF

# 启动
./build/np-client -c client.ini
```

### 查看版本

```bash
./build/np-server --version  # np-server 1.0.0
./build/np-client --version  # np-client 1.0.0
```

## 快速示例

```ini
# server.ini —— 服务端配置
[common]
bind_ip = 0.0.0.0
control_port = 7000
data_port = 7200

[kcp]
enable = 1
port = 7400

[license]
key = <your-license-key>
```

```ini
# client.ini —— 客户端配置（发布内网 SSH 与 Web 服务）
[common]
server_addr = your-server-ip
server_port = 7000
data_port = 7200
token = <node-token>

[mappings]
ssh = 127.0.0.1:22
web = 127.0.0.1:8080

[vn]
name = office-lan
token = <vn-token>
```

## 性能目标

| 指标 | 目标值 |
| --- | --- |
| 单实例并发隧道 | ≥ 50,000 |
| 单核转发吞吐 | ≥ 500 Mbps |
| 额外延迟 P99 | ≤ 5 ms |
| P2P 打洞成功率 | ≥ 80%（家庭宽带样本） |

完整指标见 [计划书 §11](docs/内网穿透项目计划书.md)。

## 开发与构建

```bash
# 数据面（C/C++17，依赖 cmake/g++）
cmake -S . -B build && cmake --build build -j
./tests/e2e.sh build          # 端到端测试

# 管理面（PHP 8.1+ webman）
cd admin && composer install && php start.php start
```

目录结构：`common/` 公共库(libnpcore) · `server/` np-server · `client/` np-client · `admin/` netportal-admin

## 路线图

- [x] M0 架构设计与协议冻结（[协议设计 v1](docs/协议设计.md)）
- [x] M1 MVP
  - [x] TCP 映射（np-server / np-client 端到端跑通）
  - [x] TLS 加密隧道（控制+数据信道，e2e 明文/TLS 双轮通过）
  - [x] UDP 映射（会话五元组管理，e2e 回显/隔离/重连通过）
  - [x] 基础控制台（webman 骨架已就绪）
- [x] M2 HTTP vhost / 限速 / 访客穿透 / KCP 加速
- [x] M3 虚拟组网（TUN/wintun、打洞、中继、子网路由）
- [x] M4 多租户 / 监控大盘 / License 系统
- [x] M5 性能调优、长稳测试、安全渗透
- [x] M6 GA 正式发布

详细排期见 [项目计划书](docs/内网穿透项目计划书.md)。

## 文档

- [项目计划书](docs/内网穿透项目计划书.md) —— 商业定位、架构设计、里程碑、风险应对
- [协议设计](docs/协议设计.md) —— 帧格式、TLV 字段、消息类型、管理面 API（含多租户 `F_TENANT`）
- [门户与动态链接配置](docs/门户与动态链接配置.md) —— Web 自助开通、隧道动态下发、多租户隔离
- [部署指南](docs/部署指南.md) —— 一键部署脚本（服务端 + 门户）、客户端安装、Nginx/HTTPS
- [域名配置指南](docs/域名配置指南.md) —— 公网服务器域名反代、伪静态、客户端 server_addr 注意事项
- [使用配置完全指南（傻瓜版）](docs/使用配置文档_傻瓜版.md) —— 零基础图文部署

## 门户控制台（多租户自助开通）

除命令行部署外，项目内置 **Web 门户**（admin 管理后端 + np-console 前端），支持：

- 用户自注册，注册时分配稳定**租户令牌**；
- 按节点配置隧道映射，保存后经 np-server **托管映射实时下发**，无需手改客户端 ini；
- 一键下载可用的 `client.ini`（已含 `tenant` 归属），客户端启动即受该租户托管；
- 服务端**强制多租户隔离**：节点首次被某租户下发后归属该租户，其他用户无法下发（返回 403），连接时声明他人租户将被拒上线。

详见 [门户与动态链接配置](docs/门户与动态链接配置.md)。

## 商业授权

本项目按**私有化部署授权**售卖：

| 规格 | 标准版 | 专业版 | 旗舰版 |
| --- | --- | --- | --- |
| 接入节点数 | ≤ 10 | ≤ 50 | 不限 |
| 总带宽 | 10 Mbps | 100 Mbps | 不限 |
| 虚拟网络数 | 1 | 5 | 不限 |
| HA 双机 | — | — | ✔ |
| 服务 | 社区问答 | 1 年维保 | 维保 + SLA + 定制 |

商务合作请通过仓库联系方式咨询。仓库内源码的开放范围以最终 LICENSE 文件为准，未经授权不得用于商业转售。

## 参与贡献

1. Fork 本仓库并新建 `feat_xxx` 分支
2. 提交遵循约定式提交：`feat:` `fix:` `docs:` `chore:`（中文摘要）
3. 发起 Pull Request，关联对应里程碑 issue

## 法律声明与免责条款

**请在使用本软件前仔细阅读本声明。下载、安装或使用本软件即视为已阅读并同意以下全部条款。**

### 1. 合规使用义务

1. 使用者应遵守所在国家/地区的法律法规。在中华人民共和国境内使用本软件的，应遵守《中华人民共和国网络安全法》《中华人民共和国数据安全法》《中华人民共和国个人信息保护法》《计算机信息网络国际联网管理暂行规定》及《互联网信息服务管理办法》等法律法规。
2. 在境内以本软件对外提供网络服务的，使用者应自行依法完成 ICP 备案、公安备案及网络安全等级保护等手续，本软件作者及版权方不承担任何备案与资质义务。
3. 使用者不得利用本软件从事下列行为：
   - 传播法律法规禁止的信息内容；
   - 未经授权访问、控制、扫描他人计算机系统或网络；
   - 绕过技术措施实施侵权、盗版、诈骗、赌博、刷单等违法活动；
   - 危害网络安全、干扰正常网络服务（包括但不限于 DDoS 攻击、流量劫持）；
   - 其他任何违反法律法规或公序良俗的行为。

### 2. 用户责任

1. 使用者对其通过本软件发布、传输、中转的全部内容和行为独立承担法律责任；
2. 使用者应妥善保管账号、Token 及授权文件，因保管不善造成的损失由使用者自行承担；
3. 企业用户建议建立内部审计制度并留存日志不少于 180 天，以满足监管核查要求。

### 3. 免责声明

1. 本软件按"现状"提供。在法律允许的最大范围内，作者及版权方不对软件的适用性、无缺陷或不中断作出明示或默示担保；
2. 因不可抗力、运营商网络故障、第三方服务变更或使用者自身原因导致的任何直接或间接损失，作者及版权方不承担责任；
3. 因使用者违反本声明或法律法规而产生的一切后果及法律责任，均由使用者自行承担；给作者、版权方或第三方造成损失的，使用者应承担赔偿责任。

### 4. 第三方组件

本软件使用了 OpenSSL、wintun、Protobuf 等第三方开源组件，相关权利归属原作者，各组件以其自身许可证条款为准。

### 5. 其他

1. 本声明的解释、效力及争议适用中华人民共和国法律（不含港澳台地区法律）；
2. 本项目为商业软件，具体许可范围以随交付物提供的正式授权协议为准；本声明与正式授权协议不一致的，以正式授权协议为准；
3. 作者保留随时修订本声明的权利，修订后将在本页面公布。

---

Copyright © 2026 NetPortal 项目组. All rights reserved.
