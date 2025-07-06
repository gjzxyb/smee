# Smee 快速参考指南

## 快速启动

### Docker Compose 方式（推荐用于开发）

```bash
# 克隆仓库
git clone https://github.com/tinkerbell/smee.git
cd smee

# 启动服务
docker compose up --build

# 停止服务
docker compose down
```

### 二进制方式

```bash
# 构建
make build

# 配置环境变量
export SMEE_BACKEND_KUBE_ENABLED=false
export SMEE_BACKEND_FILE_ENABLED=true
export SMEE_BACKEND_FILE_PATH=./test/hardware.yaml
export SMEE_OSIE_URL="http://example.com/osie/"

# 运行（需要 root 权限）
sudo -E ./cmd/smee/smee
```

## 常用命令行参数

### DHCP 配置
```bash
-dhcp-mode=reservation          # DHCP 模式：reservation/proxy/auto-proxy
-dhcp-addr=0.0.0.0:67          # DHCP 监听地址
-dhcp-enabled=true             # 启用/禁用 DHCP 服务
-dhcp-iface=eth0               # 绑定网络接口
```

### 后端配置
```bash
-backend-kube-enabled=true     # 启用 Kubernetes 后端
-backend-file-enabled=false    # 启用文件后端
-backend-file-path=./hw.yaml   # 硬件配置文件路径
-backend-noop-enabled=false    # 启用空后端（仅用于测试）
```

### HTTP/TFTP 配置
```bash
-http-addr=0.0.0.0             # HTTP 监听地址
-http-port=8080                # HTTP 监听端口
-tftp-addr=0.0.0.0             # TFTP 监听地址
-tftp-port=69                  # TFTP 监听端口
-osie-url=http://example.com/  # OSIE 镜像 URL
```

### 日志和监控
```bash
-log-level=info                # 日志级别：debug/info
-otel-endpoint=localhost:4317  # OpenTelemetry 端点
-otel-insecure=true           # 使用不安全连接
```

## 环境变量对照表

| 命令行参数 | 环境变量 | 默认值 |
|-----------|----------|--------|
| `-dhcp-addr` | `SMEE_DHCP_ADDR` | `0.0.0.0:67` |
| `-dhcp-mode` | `SMEE_DHCP_MODE` | `reservation` |
| `-backend-file-path` | `SMEE_BACKEND_FILE_PATH` | - |
| `-osie-url` | `SMEE_OSIE_URL` | - |
| `-tink-server` | `SMEE_TINK_SERVER` | - |
| `-log-level` | `SMEE_LOG_LEVEL` | `info` |

## 硬件配置文件模板

### 基本配置
```yaml
---
"aa:bb:cc:dd:ee:ff":
  ipAddress: "192.168.1.100"
  subnetMask: "255.255.255.0"
  defaultGateway: "192.168.1.1"
  nameServers:
    - "8.8.8.8"
  hostname: "server-01"
  netboot:
    allowPxe: true
```

### 完整配置
```yaml
---
"aa:bb:cc:dd:ee:ff":
  ipAddress: "192.168.1.100"
  subnetMask: "255.255.255.0"
  defaultGateway: "192.168.1.1"
  nameServers:
    - "8.8.8.8"
    - "8.8.4.4"
  hostname: "server-01"
  domainName: "example.com"
  broadcastAddress: "192.168.1.255"
  ntpServers:
    - "pool.ntp.org"
  leaseTime: 86400
  domainSearch:
    - "example.com"
  netboot:
    allowPxe: true
    osie:
      baseURL: "http://192.168.1.10/osie/"
      kernel: "vmlinuz"
      initrd: "initramfs"
```

## 常见使用场景

### 场景 1：替换现有 DHCP 服务器
```bash
# 使用预留模式
-dhcp-mode=reservation
-backend-file-enabled=true
-backend-file-path=/etc/smee/hardware.yaml
```

### 场景 2：与现有 DHCP 服务器共存
```bash
# 使用代理模式
-dhcp-mode=proxy
-backend-kube-enabled=true
```

### 场景 3：开发和测试环境
```bash
# 使用自动代理模式 + 静态脚本
-dhcp-mode=auto-proxy
-backend-noop-enabled=true
-dhcp-http-ipxe-script-url=https://boot.netboot.xyz
```

### 场景 4：仅提供 HTTP/TFTP 服务
```bash
# 禁用 DHCP
-dhcp-enabled=false
-backend-file-enabled=true
-backend-file-path=/etc/smee/hardware.yaml
```

## 故障排除检查清单

### DHCP 问题
- [ ] 检查端口 67 是否被占用：`netstat -ln | grep :67`
- [ ] 确认运行权限：需要 root 权限绑定特权端口
- [ ] 验证网络接口：`ip addr show`
- [ ] 检查防火墙规则：`iptables -L`

### 后端连接问题
- [ ] 文件后端：检查文件路径和权限
- [ ] Kubernetes 后端：验证 kubeconfig 和集群连接
- [ ] 网络连通性：`ping` 和 `telnet` 测试

### HTTP/TFTP 问题
- [ ] 检查端口监听：`netstat -ln | grep :8080`
- [ ] 验证文件权限：确保 iPXE 文件可读
- [ ] 测试 HTTP 端点：`curl http://localhost:8080/ipxe/`

### 客户端问题
- [ ] 确认 PXE 启动已启用
- [ ] 检查网络连接
- [ ] 验证 MAC 地址配置
- [ ] 查看客户端日志

## 监控检查点

### 健康检查端点
```bash
# HTTP 健康检查
curl http://localhost:8080/health

# 检查 DHCP 服务
netstat -ln | grep :67

# 检查 TFTP 服务
netstat -ln | grep :69
```

### 关键指标
- DHCP 请求成功率
- 后端查询延迟
- HTTP/TFTP 下载成功率
- 错误日志频率

### 日志位置
```bash
# 容器日志
docker logs smee

# 系统日志
journalctl -u smee

# 文件日志
tail -f /var/log/smee.log
```

## 性能调优参数

### 网络优化
```bash
-tftp-block-size=1468          # 优化 TFTP 传输
-tftp-timeout=10s              # 调整超时时间
-trusted-proxies="10.0.0.0/8"  # 配置可信代理
```

### 并发优化
```bash
# 系统级别调优
ulimit -n 65536                # 增加文件描述符限制
echo 'net.core.somaxconn = 1024' >> /etc/sysctl.conf
```

### 内存优化
```bash
# 减少内存使用
-log-level=info                # 避免 debug 级别日志
```

## 安全配置

### 网络安全
```bash
# 防火墙规则示例
iptables -A INPUT -p udp --dport 67 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -s 192.168.1.0/24 -j ACCEPT
```

### 文件权限
```bash
# 配置文件权限
chmod 600 /etc/smee/hardware.yaml
chown smee:smee /etc/smee/hardware.yaml
```

### TLS 配置
```bash
-tink-server-tls=true          # 启用 TLS
-tink-server-insecure-tls=false # 验证证书
```

## 集成配置

### Prometheus 监控
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'smee'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/metrics'
```

### Grafana 仪表板
```bash
# 导入仪表板 ID
# Smee Dashboard: 12345
```

### Kubernetes 部署
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: smee
  template:
    metadata:
      labels:
        app: smee
    spec:
      containers:
      - name: smee
        image: quay.io/tinkerbell/smee:latest
        env:
        - name: SMEE_BACKEND_KUBE_ENABLED
          value: "true"
        - name: SMEE_DHCP_MODE
          value: "proxy"
        ports:
        - containerPort: 67
          protocol: UDP
        - containerPort: 69
          protocol: UDP
        - containerPort: 8080
          protocol: TCP
```

## 常用脚本

### 健康检查脚本
```bash
#!/bin/bash
# health-check.sh

check_port() {
    local port=$1
    local protocol=${2:-tcp}
    if netstat -ln | grep -q ":$port "; then
        echo "✓ Port $port ($protocol) is listening"
        return 0
    else
        echo "✗ Port $port ($protocol) is not listening"
        return 1
    fi
}

check_port 67 udp
check_port 69 udp
check_port 8080 tcp

curl -f http://localhost:8080/health > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ HTTP health check passed"
else
    echo "✗ HTTP health check failed"
fi
```

### 配置验证脚本
```bash
#!/bin/bash
# validate-config.sh

if [ -f "$SMEE_BACKEND_FILE_PATH" ]; then
    echo "✓ Hardware config file exists"
    yamllint "$SMEE_BACKEND_FILE_PATH"
else
    echo "✗ Hardware config file not found: $SMEE_BACKEND_FILE_PATH"
fi
```

### 日志分析脚本
```bash
#!/bin/bash
# analyze-logs.sh

echo "=== DHCP Request Summary ==="
grep "received DHCP packet" /var/log/smee.log | \
    awk '{print $NF}' | sort | uniq -c

echo "=== Error Summary ==="
grep "error" /var/log/smee.log | \
    awk '{print $NF}' | sort | uniq -c
```

## 版本升级指南

### 升级前检查
```bash
# 备份配置
cp -r /etc/smee /etc/smee.backup.$(date +%Y%m%d)

# 检查当前版本
./smee -version

# 验证配置
./smee -validate-config
```

### 滚动升级
```bash
# Kubernetes 滚动升级
kubectl set image deployment/smee smee=quay.io/tinkerbell/smee:v0.x.x

# Docker Compose 升级
docker compose pull
docker compose up -d
```

---

*快速参考指南 - 最后更新：2025年1月*
