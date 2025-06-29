# 哪吒面板 K3s ArgoCD 部署

基于 GitOps 架构的哪吒面板 Kubernetes 部署配置。

## 📁 目录结构

```
apps/nezha/
├── .enabled-prod              # 生产环境启用标记文件
├── README.md                  # 本文档
├── base/                      # 基础配置
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── pvc.yaml
└── overlays/                  # 环境特定配置
    └── prod/                  # 生产环境
        ├── kustomization.yaml
        ├── ingress.yaml           # Web UI HTTP 访问
        └── ingressroute-grpc.yaml # Agent gRPC 通信
```

## 🔧 配置说明

### 基础配置 (base/)

- **镜像**：`ghcr.io/nezhahq/nezha:latest`
- **端口**：8008
- **存储**：5Gi PVC（用于存储监控数据）
- **资源**：基础资源配置，适合小型部署

### 生产环境配置 (overlays/prod/)

- **命名空间**：`nezha-prod`
- **公开访问域名**：`nezha.liuovo.com`（支持 CDN，用于 Web UI 访问）
- **Agent 通信域名**：`nezha-agent.liuovo.com`（不使用 CDN，用于 Agent 数据上报）
- **证书**：Let's Encrypt 生产证书（两个域名共用）
- **配置**：简化的生产环境配置，稳定可靠

## 🌐 域名配置说明

### 双域名架构

根据哪吒面板官方文档建议，配置了两个域名：

1. **nezha.liuovo.com**
   - 用途：Web UI 公开访问
   - 技术：标准 Ingress (HTTP/HTTPS)
   - 特点：支持 CDN 加速
   - 访问：管理员通过此域名访问面板

2. **nezha-agent.liuovo.com**
   - 用途：Agent gRPC 数据通信
   - 技术：Traefik IngressRoute (gRPC over HTTPS)
   - 特点：不使用 CDN，原生 gRPC 协议支持
   - 访问：服务器 Agent 通过此域名上报数据

### 技术实现

- **协议分离**：Web UI 使用 HTTP，Agent 使用 gRPC
- **最佳实践**：使用 Traefik IngressRoute 提供原生 gRPC 支持
- **性能优化**：HTTP/2 协议，无需额外转换层
- **证书管理**：Ingress 为两个域名申请证书，IngressRoute 复用证书



## 🚀 访问方式

- **面板管理**：https://nezha.liuovo.com
- **Agent 配置**：使用 `nezha-agent.liuovo.com:443` 作为服务器地址

## 📋 部署状态

- ✅ **生产环境已启用**：存在 `.enabled-prod` 文件
- ✅ **双域名配置**：支持 CDN 和 Agent 通信分离
- ✅ **简化配置**：稳定可靠的生产环境部署
