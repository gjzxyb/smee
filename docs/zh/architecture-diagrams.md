# Smee 架构图表和流程图

## 系统架构图

### 整体架构

```mermaid
graph TB
    subgraph "客户端层"
        C1[裸机服务器 1]
        C2[裸机服务器 2]
        C3[裸机服务器 N]
    end
    
    subgraph "网络层"
        SW[交换机/路由器]
        LB[负载均衡器]
    end
    
    subgraph "Smee 服务层"
        S1[Smee 实例 1]
        S2[Smee 实例 2]
        S3[Smee 实例 3]
    end
    
    subgraph "后端存储层"
        K8S[Kubernetes 集群]
        FILE[文件存储]
        NOOP[空后端]
    end
    
    subgraph "监控层"
        PROM[Prometheus]
        GRAF[Grafana]
        OTEL[OpenTelemetry]
    end
    
    C1 --> SW
    C2 --> SW
    C3 --> SW
    SW --> LB
    LB --> S1
    LB --> S2
    LB --> S3
    
    S1 --> K8S
    S1 --> FILE
    S1 --> NOOP
    S2 --> K8S
    S3 --> K8S
    
    S1 --> OTEL
    S2 --> OTEL
    S3 --> OTEL
    OTEL --> PROM
    PROM --> GRAF
```

### Smee 内部组件架构

```mermaid
graph TB
    subgraph "Smee 进程"
        subgraph "服务层"
            DHCP[DHCP 服务器]
            TFTP[TFTP 服务器]
            HTTP[HTTP 服务器]
            SYSLOG[Syslog 服务器]
        end
        
        subgraph "处理器层"
            RES[Reservation Handler]
            PROXY[Proxy Handler]
            SCRIPT[Script Handler]
            ISO[ISO Handler]
        end
        
        subgraph "后端层"
            KUBE[Kubernetes Backend]
            FILE_BE[File Backend]
            NOOP_BE[Noop Backend]
        end
        
        subgraph "工具层"
            METRIC[Metrics]
            OTEL_INT[OpenTelemetry]
            LOG[Logging]
        end
    end
    
    DHCP --> RES
    DHCP --> PROXY
    HTTP --> SCRIPT
    HTTP --> ISO
    
    RES --> KUBE
    RES --> FILE_BE
    PROXY --> KUBE
    PROXY --> NOOP_BE
    SCRIPT --> KUBE
    SCRIPT --> FILE_BE
    
    RES --> METRIC
    PROXY --> METRIC
    SCRIPT --> OTEL_INT
    ISO --> LOG
```

## 数据流程图

### DHCP 预留模式流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as Smee
    participant B as 后端存储
    
    Note over C,B: DHCP 发现阶段
    C->>S: DHCP Discover
    S->>B: 根据 MAC 查询硬件信息
    B->>S: 返回硬件配置
    S->>C: DHCP Offer (IP + 启动选项)
    
    Note over C,B: DHCP 请求阶段
    C->>S: DHCP Request
    S->>B: 再次查询硬件信息
    B->>S: 返回硬件配置
    S->>C: DHCP Ack (确认配置)
    
    Note over C,B: TFTP 下载阶段
    C->>S: TFTP 请求 iPXE 二进制
    S->>B: 根据 IP 查询硬件信息
    B->>S: 返回硬件配置
    S->>C: 发送 iPXE 二进制文件
    
    Note over C,B: HTTP 脚本阶段
    C->>S: HTTP 请求 iPXE 脚本
    S->>B: 根据 IP/MAC 查询硬件信息
    B->>S: 返回硬件配置
    S->>C: 发送个性化 iPXE 脚本
```

### DHCP 代理模式流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant D as 主 DHCP 服务器
    participant S as Smee (ProxyDHCP)
    participant B as 后端存储
    
    Note over C,B: 第一轮 DHCP 交换
    C->>D: DHCP Discover
    C->>S: DHCP Discover (PXE)
    D->>C: DHCP Offer (IP 地址)
    S->>B: 查询硬件信息
    B->>S: 返回硬件配置
    S->>C: DHCP Offer (启动选项)
    
    Note over C,B: 第二轮 DHCP 交换
    C->>D: DHCP Request
    C->>S: DHCP Request (PXE)
    D->>C: DHCP Ack (IP 配置)
    S->>C: DHCP Ack (启动配置)
    
    Note over C,B: 启动文件下载
    C->>S: TFTP/HTTP 请求
    S->>C: 发送启动文件
```

### iPXE 脚本生成流程

```mermaid
flowchart TD
    START([HTTP 请求到达])
    
    CHECK_MAC{从 URL 提取 MAC?}
    CHECK_IP{从 IP 提取地址?}
    
    QUERY_MAC[根据 MAC 查询后端]
    QUERY_IP[根据 IP 查询后端]
    
    FOUND_HW{找到硬件配置?}
    ALLOW_NETBOOT{允许网络启动?}
    
    STATIC_MODE{启用静态模式?}
    SERVE_STATIC[提供静态脚本]
    
    GENERATE_SCRIPT[生成个性化脚本]
    SERVE_SCRIPT[发送脚本]
    
    ERROR_404[返回 404 错误]
    
    START --> CHECK_MAC
    CHECK_MAC -->|成功| QUERY_MAC
    CHECK_MAC -->|失败| CHECK_IP
    CHECK_IP -->|成功| QUERY_IP
    CHECK_IP -->|失败| ERROR_404
    
    QUERY_MAC --> FOUND_HW
    QUERY_IP --> FOUND_HW
    
    FOUND_HW -->|是| ALLOW_NETBOOT
    FOUND_HW -->|否| STATIC_MODE
    
    ALLOW_NETBOOT -->|是| GENERATE_SCRIPT
    ALLOW_NETBOOT -->|否| ERROR_404
    
    STATIC_MODE -->|是| SERVE_STATIC
    STATIC_MODE -->|否| ERROR_404
    
    GENERATE_SCRIPT --> SERVE_SCRIPT
    SERVE_STATIC --> SERVE_SCRIPT
```

## 配置决策树

### DHCP 模式选择

```mermaid
flowchart TD
    START([开始配置])
    
    EXISTING_DHCP{现有 DHCP 服务器?}
    CONTROL_IP{需要控制 IP 分配?}
    STATIC_SCRIPT{需要静态脚本?}
    
    RESERVATION[预留模式<br/>-dhcp-mode=reservation]
    PROXY[代理模式<br/>-dhcp-mode=proxy]
    AUTO_PROXY[自动代理模式<br/>-dhcp-mode=auto-proxy]
    DISABLED[禁用 DHCP<br/>-dhcp-enabled=false]
    
    START --> EXISTING_DHCP
    
    EXISTING_DHCP -->|否| CONTROL_IP
    EXISTING_DHCP -->|是| STATIC_SCRIPT
    
    CONTROL_IP -->|是| RESERVATION
    CONTROL_IP -->|否| DISABLED
    
    STATIC_SCRIPT -->|是| AUTO_PROXY
    STATIC_SCRIPT -->|否| PROXY
```

### 后端选择决策

```mermaid
flowchart TD
    START([选择后端])
    
    ENVIRONMENT{部署环境?}
    SCALE{规模大小?}
    DYNAMIC{需要动态配置?}
    
    KUBERNETES[Kubernetes 后端<br/>-backend-kube-enabled=true]
    FILE[文件后端<br/>-backend-file-enabled=true]
    NOOP[空后端<br/>-backend-noop-enabled=true]
    
    START --> ENVIRONMENT
    
    ENVIRONMENT -->|生产环境| SCALE
    ENVIRONMENT -->|开发/测试| FILE
    ENVIRONMENT -->|演示/测试| NOOP
    
    SCALE -->|大规模| KUBERNETES
    SCALE -->|小规模| DYNAMIC
    
    DYNAMIC -->|是| KUBERNETES
    DYNAMIC -->|否| FILE
```

## 网络拓扑图

### 典型部署拓扑

```mermaid
graph TB
    subgraph "管理网络 (192.168.1.0/24)"
        MGMT_SW[管理交换机]
        SMEE_MGMT[Smee 管理接口]
        K8S_MGMT[K8s 管理接口]
    end
    
    subgraph "PXE 网络 (192.168.100.0/24)"
        PXE_SW[PXE 交换机]
        SMEE_PXE[Smee PXE 接口]
        SERVER1[服务器 1]
        SERVER2[服务器 2]
        SERVERN[服务器 N]
    end
    
    subgraph "存储网络 (192.168.200.0/24)"
        STORAGE_SW[存储交换机]
        NFS[NFS 服务器]
        ISCSI[iSCSI 存储]
    end
    
    subgraph "外部网络"
        INTERNET[互联网]
        ROUTER[路由器]
    end
    
    MGMT_SW --> SMEE_MGMT
    MGMT_SW --> K8S_MGMT
    MGMT_SW --> ROUTER
    
    PXE_SW --> SMEE_PXE
    PXE_SW --> SERVER1
    PXE_SW --> SERVER2
    PXE_SW --> SERVERN
    
    STORAGE_SW --> NFS
    STORAGE_SW --> ISCSI
    
    ROUTER --> INTERNET
    
    SMEE_MGMT -.-> SMEE_PXE
    SERVER1 -.-> STORAGE_SW
    SERVER2 -.-> STORAGE_SW
    SERVERN -.-> STORAGE_SW
```

### 高可用部署拓扑

```mermaid
graph TB
    subgraph "负载均衡层"
        LB1[负载均衡器 1]
        LB2[负载均衡器 2]
        VIP[虚拟 IP]
    end
    
    subgraph "Smee 集群"
        SMEE1[Smee 节点 1<br/>主 DHCP]
        SMEE2[Smee 节点 2<br/>备 DHCP]
        SMEE3[Smee 节点 3<br/>HTTP/TFTP]
    end
    
    subgraph "存储集群"
        ETCD1[etcd 1]
        ETCD2[etcd 2]
        ETCD3[etcd 3]
    end
    
    subgraph "客户端"
        CLIENT1[客户端 1]
        CLIENT2[客户端 2]
        CLIENTN[客户端 N]
    end
    
    VIP --> LB1
    VIP --> LB2
    
    LB1 --> SMEE1
    LB1 --> SMEE2
    LB1 --> SMEE3
    
    LB2 --> SMEE1
    LB2 --> SMEE2
    LB2 --> SMEE3
    
    SMEE1 --> ETCD1
    SMEE1 --> ETCD2
    SMEE1 --> ETCD3
    
    SMEE2 --> ETCD1
    SMEE2 --> ETCD2
    SMEE2 --> ETCD3
    
    SMEE3 --> ETCD1
    SMEE3 --> ETCD2
    SMEE3 --> ETCD3
    
    CLIENT1 --> VIP
    CLIENT2 --> VIP
    CLIENTN --> VIP
```

## 监控架构图

### 可观测性架构

```mermaid
graph TB
    subgraph "Smee 实例"
        S1[Smee 1]
        S2[Smee 2]
        S3[Smee 3]
    end
    
    subgraph "指标收集"
        OTEL[OpenTelemetry<br/>Collector]
        PROM[Prometheus]
    end
    
    subgraph "日志收集"
        FLUENTD[Fluentd]
        ELASTIC[Elasticsearch]
    end
    
    subgraph "可视化"
        GRAFANA[Grafana]
        KIBANA[Kibana]
    end
    
    subgraph "告警"
        ALERTMGR[AlertManager]
        SLACK[Slack]
        EMAIL[Email]
    end
    
    S1 --> OTEL
    S2 --> OTEL
    S3 --> OTEL
    
    S1 --> FLUENTD
    S2 --> FLUENTD
    S3 --> FLUENTD
    
    OTEL --> PROM
    FLUENTD --> ELASTIC
    
    PROM --> GRAFANA
    PROM --> ALERTMGR
    ELASTIC --> KIBANA
    
    ALERTMGR --> SLACK
    ALERTMGR --> EMAIL
```

这些图表提供了 Smee 系统的可视化表示，帮助理解其架构、数据流和部署模式。
