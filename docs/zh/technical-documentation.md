# Smee 网络启动服务技术文档

## 项目概述

Smee 是 [Tinkerbell 技术栈](https://tinkerbell.org) 中的网络启动服务，前身为 `Boots`。它是一个用 Go 语言开发的高性能网络启动服务器，专门为裸机服务器的自动化部署和管理而设计。

### 核心功能

Smee 提供以下核心服务：

1. **DHCP 服务器**
   - 仅支持主机预留（Host Reservations）
   - 基于 MAC 地址的查找机制
   - 支持网络启动选项
   - 支持 ProxyDHCP 模式
   - 多后端支持（Kubernetes、文件后端）

2. **TFTP 服务器**
   - 提供 iPXE 二进制文件服务
   - 支持可配置的块大小和超时设置

3. **HTTP 服务器**
   - 提供 iPXE 二进制文件和脚本服务
   - 基于 IP 地址的身份验证
   - 支持多后端数据源

4. **Syslog 服务器**
   - 接收并记录系统日志消息
   - 支持远程日志收集

## 架构设计

### 系统架构图

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   客户端机器     │    │      Smee       │    │     后端存储     │
│                │    │                │    │                │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  ┌───────────┐  │
│  │ DHCP 客户端│◄─┼────┼─►│DHCP 服务器 │  │    │  │Kubernetes │  │
│  └───────────┘  │    │  └───────────┘  │    │  │    或     │  │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  │ 文件后端  │  │
│  │TFTP 客户端│◄─┼────┼─►│TFTP 服务器 │◄─┼────┼─►│           │  │
│  └───────────┘  │    │  └───────────┘  │    │  └───────────┘  │
│  ┌───────────┐  │    │  ┌───────────┐  │    └─────────────────┘
│  │HTTP 客户端│◄─┼────┼─►│HTTP 服务器 │  │
│  └───────────┘  │    │  └───────────┘  │
└─────────────────┘    │  ┌───────────┐  │
                       │  │Syslog服务器│  │
                       │  └───────────┘  │
                       └─────────────────┘
```

### 核心组件

#### 1. DHCP 处理器

**预留模式处理器 (Reservation Handler)**
- 位置：`internal/dhcp/handler/reservation/`
- 功能：处理基于主机预留的 DHCP 请求
- 特点：
  - 仅响应已配置 MAC 地址的设备
  - 提供固定 IP 地址分配
  - 支持网络启动选项配置

**代理模式处理器 (Proxy Handler)**
- 位置：`internal/dhcp/handler/proxy/`
- 功能：作为 ProxyDHCP 服务器运行
- 特点：
  - 不分配 IP 地址，仅提供启动信息
  - 与现有 DHCP 服务器协同工作
  - 支持自动代理模式

#### 2. 后端存储系统

**Kubernetes 后端**
- 位置：`internal/backend/kube/`
- 功能：从 Kubernetes 集群获取硬件配置
- 特点：
  - 使用 controller-runtime 客户端
  - 支持 MAC 地址和 IP 地址索引
  - 实时数据同步

**文件后端**
- 位置：`internal/backend/file/`
- 功能：从 YAML 文件读取硬件配置
- 特点：
  - 支持文件监控和热重载
  - 简单的配置管理
  - 适合开发和测试环境

#### 3. iPXE 脚本服务

**脚本处理器**
- 位置：`internal/ipxe/script/`
- 功能：生成和提供 iPXE 启动脚本
- 特点：
  - 支持动态脚本生成
  - 基于硬件配置的个性化脚本
  - 支持静态脚本模式

## DHCP 工作模式

### 1. 预留模式 (Reservation Mode)
```bash
-dhcp-mode=reservation
```
- **默认模式**
- 响应 DHCP 请求并提供 IP 地址和启动信息
- 所有 IP 信息基于预留配置
- 必须存在对应的硬件记录

### 2. 代理模式 (Proxy Mode)
```bash
-dhcp-mode=proxy
```
- 响应支持 PXE 的 DHCP 请求
- 仅提供启动信息，不分配 IP 地址
- 需要现有 DHCP 服务器提供 IP 分配
- 必须存在对应的硬件记录

### 3. 自动代理模式 (Auto Proxy Mode)
```bash
-dhcp-mode=auto-proxy
```
- 类似代理模式，但更灵活
- 如果找不到硬件记录，提供静态 iPXE 脚本
- 如果找到硬件记录，提供正常的 auto.ipxe 脚本
- 可配合 `-backend-noop-enabled` 禁用后端查找

### 4. DHCP 禁用模式
```bash
-dhcp-enabled=false
```
- 不响应 DHCP 请求
- 仅提供 TFTP 和 HTTP 功能
- 适合已有 DHCP 服务器的环境

## 配置管理

### 环境变量与命令行参数

所有命令行参数都可以通过环境变量设置：
- 环境变量前缀：`SMEE_`
- 全部大写
- 连字符替换为下划线

示例：
```bash
# 命令行参数
-dhcp-addr=0.0.0.0:67

# 对应环境变量
SMEE_DHCP_ADDR=0.0.0.0:67
```

### 关键配置参数

**DHCP 配置**
```bash
-dhcp-addr                    # DHCP 监听地址 (默认: 0.0.0.0:67)
-dhcp-mode                    # DHCP 模式 (默认: reservation)
-dhcp-iface                   # 绑定网络接口
-dhcp-ip-for-packet          # DHCP 包中使用的 IP 地址
```

**后端配置**
```bash
-backend-kube-enabled         # 启用 Kubernetes 后端 (默认: true)
-backend-file-enabled         # 启用文件后端 (默认: false)
-backend-file-path           # 硬件配置文件路径
```

**HTTP 配置**
```bash
-http-addr                    # HTTP 监听地址
-http-port                    # HTTP 监听端口 (默认: 8080)
-osie-url                     # OSIE 镜像 URL
-tink-server                  # Tink 服务器地址
```

## 数据流程

### 典型启动流程

1. **DHCP 发现阶段**
   ```
   客户端 → Smee: DHCP Discover
   Smee → 后端: 根据 MAC 获取硬件数据
   后端 → Smee: 返回硬件数据
   Smee → 客户端: DHCP Offer
   ```

2. **DHCP 请求阶段**
   ```
   客户端 → Smee: DHCP Request
   Smee → 后端: 根据 MAC 获取硬件数据
   后端 → Smee: 返回硬件数据
   Smee → 客户端: DHCP Ack
   ```

3. **TFTP 下载阶段**
   ```
   客户端 → Smee: TFTP 获取 iPXE 二进制
   Smee → 后端: 根据 IP 获取硬件数据
   后端 → Smee: 返回硬件数据
   Smee → 客户端: 发送 iPXE 二进制
   ```

4. **HTTP 脚本阶段**
   ```
   客户端 → Smee: HTTP 获取 iPXE 脚本
   Smee → 后端: 根据 IP 获取硬件数据
   后端 → Smee: 返回硬件数据
   Smee → 客户端: 发送 iPXE 脚本
   ```

## 开发指南

### 代码结构

```
cmd/smee/           # 主程序入口
internal/
├── backend/        # 后端存储实现
│   ├── file/      # 文件后端
│   └── kube/      # Kubernetes 后端
├── dhcp/          # DHCP 服务实现
│   ├── handler/   # DHCP 处理器
│   └── server/    # DHCP 服务器
├── ipxe/          # iPXE 相关功能
│   ├── http/      # HTTP 服务
│   └── script/    # 脚本生成
├── iso/           # ISO 镜像处理
├── metric/        # 监控指标
├── otel/          # OpenTelemetry 集成
└── syslog/        # Syslog 服务
```

### 设计原则

1. **简单易懂优于简单易用**
2. **先实现，再优化，再改进，最后测试**
3. **明确 Goroutine 的生命周期**
4. **避免包级别和全局变量**
5. **接受接口，返回结构体**
6. **小接口更好**

### 依赖管理

项目使用 Go Modules 进行依赖管理：
- Go 版本：1.24.0+
- 主要依赖：
  - `github.com/insomniacslk/dhcp` - DHCP 协议实现
  - `github.com/tinkerbell/ipxedust` - iPXE 二进制服务
  - `k8s.io/client-go` - Kubernetes 客户端
  - `go.opentelemetry.io/otel` - 可观测性

## 部署指南

### Docker Compose 部署

```bash
# 构建并启动服务
docker compose up --build

# 停止服务
docker compose down
```

### 本地开发部署

```bash
# 构建二进制文件
make build

# 设置环境变量
export SMEE_OSIE_URL=<OSIE 镜像 URL>
export SMEE_BACKEND_KUBE_ENABLED=false
export SMEE_BACKEND_FILE_ENABLED=true
export SMEE_BACKEND_FILE_PATH=./test/hardware.yaml
export SMEE_EXTRA_KERNEL_ARGS="tink_worker_image=quay.io/tinkerbell/tink-worker:latest"

# 运行服务（需要 root 权限绑定低端口）
sudo -E ./cmd/smee/smee
```

### Kubernetes 部署

参考 `docs/manifests/` 目录下的部署清单文件。

## 监控与可观测性

### OpenTelemetry 集成

Smee 内置 OpenTelemetry 支持：
```bash
-otel-endpoint          # OpenTelemetry 收集器端点
-otel-insecure         # 使用不安全连接 (默认: true)
```

### 监控指标

- DHCP 请求处理指标
- HTTP 请求指标
- 后端查询性能指标
- 错误率统计

### 日志记录

支持结构化日志记录：
```bash
-log-level             # 日志级别 (debug, info) (默认: info)
```

## 故障排除

### 常见问题

1. **DHCP 服务无法启动**
   - 检查端口 67 是否被占用
   - 确认运行权限（需要 root 权限）
   - 验证网络接口配置

2. **客户端无法获取 IP 地址**
   - 检查硬件配置文件中是否存在对应 MAC 地址
   - 验证 DHCP 模式配置
   - 检查网络连通性

3. **iPXE 脚本下载失败**
   - 验证 HTTP 服务器配置
   - 检查后端数据源连接
   - 确认客户端 IP 地址匹配

### 调试技巧

1. **启用调试日志**
   ```bash
   -log-level=debug
   ```

2. **检查网络包**
   ```bash
   tcpdump -i <interface> port 67 or port 69 or port 8080
   ```

3. **验证配置文件**
   ```bash
   # 检查 YAML 语法
   yamllint test/hardware.yaml
   ```

## 扩展开发

### 添加新的后端

1. 实现 `handler.BackendReader` 接口
2. 在 `internal/backend/` 下创建新包
3. 在主配置中注册新后端

### 自定义 iPXE 脚本

1. 修改 `internal/ipxe/script/` 中的模板
2. 添加新的脚本生成逻辑
3. 更新路由处理器

### 添加新的监控指标

1. 在 `internal/metric/` 中定义新指标
2. 在相应组件中记录指标数据
3. 配置 OpenTelemetry 导出

## 贡献指南

1. Fork 项目仓库
2. 创建功能分支
3. 编写测试用例
4. 提交 Pull Request
5. 通过代码审查

### 代码规范

- 遵循 Go 官方代码规范
- 使用 `gofmt` 格式化代码
- 编写单元测试
- 添加适当的文档注释

## API 接口文档

### HTTP 端点

#### iPXE 脚本端点

**GET /auto.ipxe**
- 功能：获取自动生成的 iPXE 启动脚本
- 认证：基于客户端 IP 地址
- 响应：iPXE 脚本内容

**GET /{mac}/auto.ipxe**
- 功能：根据 MAC 地址获取特定的 iPXE 脚本
- 参数：`mac` - 硬件 MAC 地址
- 响应：个性化的 iPXE 脚本

**GET /custom.ipxe**
- 功能：获取自定义 iPXE 脚本
- 认证：基于客户端 IP 地址
- 响应：自定义脚本内容

#### iPXE 二进制端点

**GET /ipxe/{filename}**
- 功能：下载 iPXE 二进制文件
- 参数：`filename` - iPXE 二进制文件名
- 支持的文件：
  - `undionly.kpxe` - 传统 BIOS 网络启动
  - `ipxe.efi` - UEFI 网络启动
  - `snp.efi` - UEFI SNP 驱动启动

#### ISO 镜像端点

**GET /iso/**
- 功能：获取修补后的 ISO 镜像
- 特点：动态修补内核参数
- 用途：静态 IPAM 配置

### DHCP 选项说明

#### 标准 DHCP 选项

| 选项 | 名称 | 描述 |
|------|------|------|
| 1 | 子网掩码 | 客户端网络子网掩码 |
| 3 | 路由器 | 默认网关地址 |
| 6 | 域名服务器 | DNS 服务器地址列表 |
| 7 | 日志服务器 | Syslog 服务器地址 |
| 12 | 主机名 | 客户端主机名 |
| 15 | 域名 | 客户端域名 |
| 28 | 广播地址 | 网络广播地址 |
| 42 | NTP 服务器 | 时间同步服务器 |
| 51 | 租约时间 | IP 地址租约时长 |
| 119 | 域搜索 | 域名搜索列表 |

#### PXE 专用选项

| 选项 | 名称 | 描述 |
|------|------|------|
| 43 | 厂商特定信息 | PXE 启动控制信息 |
| 54 | 服务器标识符 | DHCP 服务器 IP 地址 |
| 60 | 厂商类标识符 | 客户端类型标识 |
| 66 | TFTP 服务器名 | TFTP 服务器地址 |
| 67 | 启动文件名 | 启动文件路径 |
| 77 | 用户类 | 用户类别信息 |
| 97 | 客户端机器标识符 | 客户端唯一标识 |

## 配置文件详解

### 硬件配置文件格式

```yaml
# hardware.yaml 示例
---
"02:00:00:00:00:ff":           # MAC 地址（必须用引号）
  ipAddress: "192.168.99.43"   # 分配的 IP 地址
  subnetMask: "255.255.255.0"  # 子网掩码
  defaultGateway: "192.168.99.1" # 默认网关
  nameServers:                 # DNS 服务器列表
    - "8.8.8.8"
    - "8.8.4.4"
  hostname: "server-01"        # 主机名
  domainName: "example.com"    # 域名
  broadcastAddress: "192.168.99.255" # 广播地址
  ntpServers:                  # NTP 服务器列表
    - "pool.ntp.org"
  leaseTime: 86400            # 租约时间（秒）
  domainSearch:               # 域搜索列表
    - "example.com"
    - "local.example.com"
  netboot:                    # 网络启动配置
    allowPxe: true            # 是否允许 PXE 启动
    osie:                     # OSIE 配置
      baseURL: "http://example.com/osie/"
      kernel: "vmlinuz"
      initrd: "initramfs"
    ipxe:                     # iPXE 配置
      scriptURL: "http://example.com/custom.ipxe"
```

### Kubernetes 硬件资源定义

```yaml
apiVersion: tinkerbell.org/v1alpha1
kind: Hardware
metadata:
  name: server-01
  namespace: tinkerbell-system
spec:
  interfaces:
    - dhcp:
        mac: "02:00:00:00:00:ff"
        ip:
          address: "192.168.99.43"
          netmask: "255.255.255.0"
          gateway: "192.168.99.1"
        nameservers:
          - "8.8.8.8"
        timeservers:
          - "pool.ntp.org"
        hostname: "server-01"
        domainName: "example.com"
        leaseTime: 86400
        domainSearch:
          - "example.com"
      netboot:
        allowPXE: true
        allowWorkflow: true
        ipxe:
          url: "http://192.168.99.42:8080/auto.ipxe"
          contents: |
            #!ipxe
            echo Starting custom iPXE script
            dhcp
            chain http://example.com/boot.ipxe
```

## 性能优化

### 网络性能优化

1. **TFTP 块大小调优**
   ```bash
   -tftp-block-size=1468  # 接近 MTU 大小，减少分片
   ```

2. **HTTP 连接优化**
   ```bash
   -trusted-proxies="10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
   ```

3. **并发连接控制**
   - 使用连接池管理数据库连接
   - 限制并发 DHCP 请求处理数量

### 内存优化

1. **文件后端缓存**
   - 使用内存缓存减少文件 I/O
   - 实现文件变更监控和热重载

2. **Kubernetes 后端优化**
   - 使用客户端缓存减少 API 调用
   - 配置合适的缓存过期时间

### 存储优化

1. **日志轮转**
   ```bash
   # 配置日志轮转策略
   logrotate /var/log/smee.log
   ```

2. **监控数据清理**
   - 定期清理过期的监控数据
   - 配置合适的数据保留策略

## 安全考虑

### 网络安全

1. **访问控制**
   - 使用防火墙限制访问端口
   - 配置网络 ACL 规则

2. **通信加密**
   ```bash
   -tink-server-tls=true  # 启用 TLS 通信
   ```

3. **IP 白名单**
   - 在硬件配置中明确指定允许的 IP 地址
   - 使用网络分段隔离 PXE 网络

### 数据安全

1. **配置文件保护**
   ```bash
   chmod 600 hardware.yaml  # 限制文件访问权限
   ```

2. **敏感信息管理**
   - 使用 Kubernetes Secrets 存储敏感配置
   - 避免在日志中记录敏感信息

3. **审计日志**
   - 记录所有 DHCP 请求和响应
   - 监控异常访问模式

## 高可用部署

### 负载均衡配置

1. **DHCP 高可用**
   ```bash
   # 使用 DHCP 故障转移配置
   # 主服务器
   -dhcp-mode=reservation

   # 备用服务器
   -dhcp-mode=proxy
   ```

2. **HTTP 服务负载均衡**
   ```yaml
   # nginx 配置示例
   upstream smee_backend {
       server 192.168.1.10:8080;
       server 192.168.1.11:8080;
       server 192.168.1.12:8080;
   }
   ```

### 数据同步

1. **文件后端同步**
   ```bash
   # 使用 rsync 同步配置文件
   rsync -av hardware.yaml backup-server:/etc/smee/
   ```

2. **Kubernetes 后端**
   - 利用 Kubernetes 原生的高可用特性
   - 配置多个 API 服务器端点

## 测试指南

### 单元测试

```bash
# 运行所有测试
make test

# 运行特定包的测试
go test ./internal/dhcp/handler/reservation/

# 运行测试并生成覆盖率报告
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### 集成测试

```bash
# 使用 Docker Compose 运行集成测试
docker compose -f docker-compose.test.yml up --build

# 手动集成测试
./test/test-smee.sh
```

### 测试用例结构

```go
// 示例测试用例
func TestDHCPHandler(t *testing.T) {
    tests := []struct {
        name     string
        input    *dhcpv4.DHCPv4
        expected *dhcpv4.DHCPv4
        wantErr  bool
    }{
        {
            name: "valid DHCP discover",
            input: &dhcpv4.DHCPv4{
                OpCode:       dhcpv4.OpcodeBootRequest,
                HWType:       iana.HWTypeEthernet,
                ClientHWAddr: net.HardwareAddr{0x02, 0x00, 0x00, 0x00, 0x00, 0xff},
            },
            expected: &dhcpv4.DHCPv4{
                OpCode:    dhcpv4.OpcodeBootReply,
                YourIPAddr: net.ParseIP("192.168.99.43"),
            },
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 测试逻辑
        })
    }
}
```

## 与 Tinkerbell 生态系统集成

### Tink 工作流集成

```yaml
# 工作流模板示例
apiVersion: tinkerbell.org/v1alpha1
kind: Template
metadata:
  name: ubuntu-installation
spec:
  data: |
    version: "0.1"
    name: ubuntu_provisioning
    global_timeout: 6000
    tasks:
      - name: "os-installation"
        worker: "{{.device_1}}"
        volumes:
          - /dev:/dev
          - /dev/console:/dev/console
          - /lib/firmware:/lib/firmware:ro
        actions:
          - name: "stream-ubuntu-image"
            image: quay.io/tinkerbell-actions/image2disk:v1.0.0
            timeout: 600
            environment:
              DEST_DISK: /dev/sda
              IMG_URL: "http://192.168.1.100/ubuntu-20.04.img"
              COMPRESSED: true
```

### Hook OS 集成

```bash
# 配置 Hook OS 参数
export SMEE_OSIE_URL="http://192.168.1.100/hook/"
export SMEE_EXTRA_KERNEL_ARGS="tink_worker_image=quay.io/tinkerbell/tink-worker:latest console=tty0 console=ttyS0,115200"
```

### Hegel 元数据服务集成

```yaml
# 硬件元数据配置
apiVersion: tinkerbell.org/v1alpha1
kind: Hardware
metadata:
  name: server-01
spec:
  metadata:
    facility:
      facility_code: "dc1"
      plan_slug: "c3.medium.x86"
    instance:
      hostname: "server-01"
      id: "12345678-1234-1234-1234-123456789012"
    state: "provisioning"
```

## 监控和告警

### Prometheus 指标

```go
// 自定义指标示例
var (
    dhcpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "smee_dhcp_requests_total",
            Help: "Total number of DHCP requests processed",
        },
        []string{"message_type", "status"},
    )

    backendQueryDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "smee_backend_query_duration_seconds",
            Help: "Duration of backend queries",
        },
        []string{"backend_type", "operation"},
    )
)
```

### Grafana 仪表板

```json
{
  "dashboard": {
    "title": "Smee Network Boot Service",
    "panels": [
      {
        "title": "DHCP Requests Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(smee_dhcp_requests_total[5m])",
            "legendFormat": "{{message_type}} - {{status}}"
          }
        ]
      },
      {
        "title": "Backend Query Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(smee_backend_query_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      }
    ]
  }
}
```

### 告警规则

```yaml
# Prometheus 告警规则
groups:
  - name: smee.rules
    rules:
      - alert: SmeeHighErrorRate
        expr: rate(smee_dhcp_requests_total{status="error"}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Smee DHCP error rate is high"
          description: "DHCP error rate is {{ $value }} requests per second"

      - alert: SmeeBackendDown
        expr: up{job="smee"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Smee service is down"
          description: "Smee service has been down for more than 1 minute"
```

## 最佳实践

### 网络设计

1. **网络分段**
   - 将 PXE 网络与生产网络分离
   - 使用 VLAN 隔离不同环境

2. **IP 地址规划**
   ```
   管理网络：  192.168.1.0/24
   PXE 网络：  192.168.100.0/24
   存储网络：  192.168.200.0/24
   ```

3. **DHCP 中继配置**
   ```bash
   # Cisco 交换机配置示例
   interface vlan100
     ip helper-address 192.168.1.10  # Smee 服务器地址
   ```

### 容量规划

1. **硬件要求**
   ```
   CPU：    2-4 核心（支持 100+ 并发启动）
   内存：   4-8 GB（取决于硬件配置数量）
   存储：   50-100 GB（用于镜像和日志）
   网络：   1 Gbps（支持快速镜像下载）
   ```

2. **性能基准**
   ```
   DHCP 处理：    1000+ 请求/秒
   TFTP 传输：    50+ 并发连接
   HTTP 下载：    100+ 并发连接
   ```

### 运维建议

1. **日志管理**
   ```bash
   # 配置日志轮转
   /var/log/smee/*.log {
       daily
       rotate 30
       compress
       delaycompress
       missingok
       notifempty
       create 644 smee smee
   }
   ```

2. **备份策略**
   ```bash
   # 备份配置文件
   tar -czf smee-config-$(date +%Y%m%d).tar.gz \
       /etc/smee/ \
       /var/lib/smee/
   ```

3. **健康检查**
   ```bash
   #!/bin/bash
   # health-check.sh

   # 检查 DHCP 端口
   if ! netstat -ln | grep -q ":67 "; then
       echo "DHCP port not listening"
       exit 1
   fi

   # 检查 HTTP 端口
   if ! curl -f http://localhost:8080/health; then
       echo "HTTP health check failed"
       exit 1
   fi

   echo "All checks passed"
   ```

## 社区和支持

### 获取帮助

1. **官方文档**
   - [Tinkerbell 官网](https://tinkerbell.org)
   - [GitHub 仓库](https://github.com/tinkerbell/smee)

2. **社区支持**
   - Slack: #tinkerbell
   - GitHub Issues
   - 社区论坛

3. **商业支持**
   - 联系 Tinkerbell 团队
   - 企业级支持服务

### 贡献代码

1. **开发流程**
   ```bash
   # Fork 仓库
   git clone https://github.com/your-username/smee.git

   # 创建功能分支
   git checkout -b feature/new-feature

   # 提交更改
   git commit -m "Add new feature"

   # 推送分支
   git push origin feature/new-feature

   # 创建 Pull Request
   ```

2. **代码审查**
   - 所有代码必须通过审查
   - 确保测试覆盖率 > 80%
   - 遵循项目编码规范

## 许可证

本项目采用 Apache 2.0 许可证，详见 LICENSE 文件。

---

*本文档最后更新时间：2025年1月*
