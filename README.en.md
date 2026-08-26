# NetPortal Intranet Penetration System

> A privately deployable NAT traversal platform with **both port forwarding** and **overlay virtual networking**.
> Management plane built on webman (PHP), data plane implemented in high-performance C/C++.

[![License](https://img.shields.io/badge/License-Commercial-blue)](#commercial-licensing)
[![Version](https://img.shields.io/badge/Version-1.0.0-green)](#changelog)
[![Status](https://img.shields.io/badge/Status-GA-blue)](#roadmap)

[简体中文](README.md) | [English](README.en.md)

---

## Introduction

NetPortal solves the problem of intranet devices and services being unreachable behind NAT / firewalls, offering two complementary capabilities:

| Mode | Typical Scenarios |
| --- | --- |
| **Port Forwarding**: publish intranet services on public server ports | Remote access to NAS, CCTV, ERP, self-hosted websites; exposing APIs |
| **Virtual Networking (Overlay VPN)**: join devices into one virtual LAN | Multi-site office interconnection, chain-store networking, industrial remote maintenance |

Unlike SaaS tunneling services, NetPortal targets **private deployment sales**: all traffic flows through the customer's own servers, with full control over console, licensing, and monitoring.

## Key Features

### Port Forwarding
- TCP / UDP service publishing, HTTP(S) domain-based reverse proxy (vhost)
- Full TLS encrypted tunnel, optional end-to-end secret tunnel (visitor mode)
- KCP/QUIC acceleration for lossy links, compression, P2P hole punching
- Two-level bandwidth/connection limits per rule and per node

### Virtual Networking
- Multiple isolated virtual networks (VN), automatic virtual IP allocation
- TUN / wintun adapters across Linux / Windows / macOS / ARM
- P2P direct connection via NAT hole punching, automatic relay fallback
- Subnet routing: share an entire physical subnet with VN members

### Management Console (webman)
- Multi-tenant RBAC, node lifecycle management, hot config reload
- Real-time traffic monitoring, audit logs, OpenAPI integration
- License management (machine fingerprint binding, offline activation)

## Architecture

```
                 ┌────────────────────────────────────────────┐
                 │           Management Plane (webman)         │
  Browser ──────▶│  Console(Vue3) + Admin API + License Check  │
  3rd-party ────▶│  OpenAPI                                    │
                 └───────┬─────────────────┬──────────────────┘
                         │ MySQL           │ Redis
                         ▼                 ▼
┌────────────────┐  Control/Config  ┌──────────────────────┐
│ np-client      │◀═══TLS channel═▶ │ np-server (public)    │
│ (intranet host)│                  │  - Tunnel gateway     │◀── Visitors
│  - Port maps   │═══data channel═▶ │  - vhost proxy        │    (HTTP/TCP/UDP)
│  - Virtual NIC │   TCP/UDP/KCP    │  - P2P signaling/relay│◀── VN members
└───────┬────────┘                  └──────────┬───────────┘
        │ TUN/wintun                           │ STUN
        ▼                                      ▼
   LAN services                          P2P direct links
```

| Component | Language | Description |
| --- | --- | --- |
| netportal-admin | PHP (webman) | Admin API, console backend, tenants & licensing |
| np-server | C/C++ | Public tunnel gateway: forwarding, vhost proxy, P2P signaling & relay |
| np-client | C/C++ | Intranet client: port mapping, virtual NIC, subnet routing |
| np-console | Vue3 + TS | Web console frontend |

## Tech Stack

- **Data plane**: C++17 · epoll/IOCP (io_uring-ready) · OpenSSL/mbedTLS · Protobuf(nanopb) · wintun/TUN
- **Management plane**: webman (workerman) · MySQL 8 · Redis 7
- **Frontend**: Vue 3 · TypeScript · ECharts
- **Build**: CMake with cross-compilation for embedded ARM

## Installation & Usage

### Build from Source

```bash
# Dependencies: cmake >= 3.16, g++ (C++17), OpenSSL development headers
# Debian/Ubuntu: apt install cmake g++ libssl-dev
# CentOS/RHEL:   dnf install cmake gcc-c++ openssl-devel

cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)

# Binaries
ls build/np-server build/np-client
```

### Server Deployment (public machine)

```bash
# Create config file
cat > server.ini <<'EOF'
[common]
bind_ip = 0.0.0.0
control_port = 7000
data_port = 7200

[license]
key = <your-license-key>
EOF

# Start
./build/np-server -c server.ini
```

### Client Deployment (intranet machine)

```bash
# Create config file
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

# Start
./build/np-client -c client.ini
```

### Check Version

```bash
./build/np-server --version  # np-server 1.0.0
./build/np-client --version  # np-client 1.0.0
```

## Quick Example

```ini
# server.ini —— Server configuration
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
# client.ini —— Client configuration (publish intranet SSH & web services)
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

## Performance Targets

| Metric | Target |
| --- | --- |
| Concurrent tunnels per instance | ≥ 50,000 |
| Single-core forwarding throughput | ≥ 500 Mbps |
| Added latency P99 | ≤ 5 ms |
| P2P hole-punch success rate | ≥ 80% (residential broadband sample) |

See the [project plan §11](docs/内网穿透项目计划书.md) for full details.

## Roadmap

- [x] M0 Architecture design & protocol freeze
- [x] M1 MVP: TCP/UDP mapping + encrypted tunnel + basic console
- [x] M2 HTTP vhost / rate limiting / visitor tunnels / KCP acceleration
- [x] M3 Virtual networking (TUN/wintun, hole punching, relay, subnet routing)
- [x] M4 Multi-tenant / monitoring dashboard / license system
- [x] M5 Performance tuning, endurance testing, security audit
- [x] M6 GA release

Detailed schedule in the [Project Plan (Chinese)](docs/内网穿透项目计划书.md).

## Documentation

- [项目计划书 / Project Plan](docs/内网穿透项目计划书.md) —— business positioning, architecture design, milestones, risk management

## Commercial Licensing

NetPortal is sold as a **privately deployed** product:

| Spec | Standard | Professional | Enterprise |
| --- | --- | --- | --- |
| Nodes | ≤ 10 | ≤ 50 | Unlimited |
| Bandwidth | 10 Mbps | 100 Mbps | Unlimited |
| Virtual networks | 1 | 5 | Unlimited |
| HA pair | — | — | ✔ |
| Support | Community | 1-year maintenance | Maintenance + SLA + custom dev |

Contact us through repository channels for business inquiries. Source availability and redistribution rights are governed by the final LICENSE file; unauthorized commercial resale is prohibited.

## Contributing

1. Fork this repository and create a `feat_xxx` branch
2. Follow conventional commits: `feat:` `fix:` `docs:` `chore:` (Chinese summary)
3. Open a Pull Request linked to the milestone issue

## Legal Notice & Disclaimer

**Please read this notice carefully before using the software. Downloading, installing, or using the software constitutes acceptance of all terms below.**

### 1. Compliance Obligations

1. Users shall comply with applicable laws and regulations of their jurisdiction. Within the People's Republic of China, users shall comply with the Cybersecurity Law, Data Security Law, Personal Information Protection Law, Regulations on the Administration of International Networking of Computer Information Networks, and related regulations.
2. Users providing network services in mainland China with this software are solely responsible for ICP filing, public security filing, classified protection of cybersecurity (MLPS), and other statutory procedures. Neither the authors nor the copyright holder bears any obligation for such filings or qualifications.
3. Users must NOT use this software to:
   - Distribute content prohibited by laws and regulations;
   - Access, control, or scan computer systems or networks without authorization;
   - Conduct piracy, fraud, gambling, click farming, or other illegal activities by circumventing technical measures;
   - Endanger network security or disrupt legitimate services (including but not limited to DDoS attacks, traffic hijacking);
   - Engage in any other unlawful or improper activities.

### 2. User Responsibility

1. Users bear full legal responsibility for all content transmitted and actions performed through this software;
2. Users shall safeguard accounts, tokens, and license files; losses caused by poor custody shall be borne by users themselves;
3. Enterprise users are advised to establish internal audit systems and retain logs for no less than 180 days to satisfy regulatory requirements.

### 3. Disclaimer of Warranty

1. This software is provided "as is". To the maximum extent permitted by law, the authors and copyright holder make no express or implied warranties of merchantability, fitness for a particular purpose, or non-infringement;
2. The authors and copyright holder shall not be liable for any direct or indirect losses caused by force majeure, carrier network failures, third-party service changes, or user's own causes;
3. All consequences and legal liabilities arising from users' violation of this notice or applicable laws shall be borne solely by users; users shall indemnify the authors, copyright holder, or third parties against resulting damages.

### 4. Third-party Components

This software uses third-party open-source components including OpenSSL, wintun, and Protobuf. Rights belong to their respective authors and each component is governed by its own license.

### 5. Miscellaneous

1. This notice shall be governed by the laws of the People's Republic of China (excluding Hong Kong, Macao, and Taiwan);
2. This project is commercial software; the specific scope of licensing is subject to the formal license agreement provided with the deliverables. In case of conflict between this notice and the formal agreement, the latter prevails;
3. The authors reserve the right to revise this notice at any time; revisions will be published on this page.

---

Copyright © 2026 NetPortal Project Team. All rights reserved.
