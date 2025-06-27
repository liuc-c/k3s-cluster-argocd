# K3s Cluster GitOps with ArgoCD

> [!IMPORTANT]
> **个人项目声明**
>
> 这是一个**个人学习和自用**的 K3s GitOps 实践项目。本项目仅供参考和学习使用。
>
> ⚠️ **使用前请注意**：
> - 项目中的域名、证书配置等均为个人环境配置
> - 在使用前请根据您的实际环境调整所有相关配置
> - 建议先在测试环境中验证配置的正确性
> - 作者不对使用本配置造成的任何问题承担责任
>
> 📚 **适用场景**：个人学习 GitOps、K3s 集群管理、ArgoCD 应用部署等技术实践

---

这是一个基于 K3s 集群的 GitOps 实践项目，使用 ArgoCD + Kustomize 实现应用的自动化部署和管理。

## 🏗️ 架构概览

- **集群**: K3s (轻量级 Kubernetes)
- **GitOps 工具**: ArgoCD
- **配置管理**: Kustomize (Base + Overlays 模式)
- **入口控制器**: Traefik
- **证书管理**: cert-manager + Let's Encrypt
- **域名管理**: 自定义域名 + 自动 TLS 证书

## 📁 项目结构

```
k3s-cluster-argocd/
├── README.md                     # 项目文档
├── argocd/                       # ArgoCD 配置
│   ├── apps/                     # ArgoCD 应用定义
│   │   ├── uptime-kuma-prod-app.yaml
│   │   ├── vaultwarden-prod-app.yaml.disable
│   │   └── monitoring-prod-app.yaml.disable
│   └── config/                   # ArgoCD 核心配置
│       ├── README.md
│       └── argocd-cm-patch.yaml
└── apps/                         # 应用配置
    ├── uptime-kuma/              # 服务监控
    │   ├── base/                 # 基础配置
    │   └── overlays/prod/        # 生产环境覆盖
    ├── vaultwarden/              # 密码管理器
    │   ├── base/
    │   └── overlays/prod/
    └── monitoring/               # 监控堆栈
        └── prod/
```

## 🚀 已部署的应用

### 生产环境应用

| 应用名称 | 描述 | 访问方式 | 状态 |
|---------|------|---------|------|
| **uptime-kuma** | 服务监控和状态页面 | HTTPS + 自定义域名 | ✅ 活跃 |
| **vaultwarden** | 开源密码管理器 | HTTPS + 自定义域名 | 🔒 已禁用 |
| **monitoring** | Prometheus + Grafana 监控堆栈 | HTTPS + 自定义域名 | 🔒 已禁用 |

### 应用详情

#### 🔍 Uptime Kuma
- **功能**: 网站和服务的可用性监控
- **镜像**: `louislam/uptime-kuma:1`
- **存储**: 2GB 持久化存储
- **特性**:
  - 实时状态监控
  - 多种通知方式
  - 美观的状态页面
  - 支持多种协议检查

#### 🔐 Vaultwarden (已禁用)
- **功能**: Bitwarden 兼容的密码管理器
- **镜像**: `vaultwarden/server:latest`
- **存储**: 1GB 持久化存储

#### 📊 Monitoring Stack (已禁用)
- **功能**: 集群监控和可观测性
- **组件**: Prometheus + Grafana + AlertManager
- **版本**: kube-prometheus-stack 75.6.0

## 🛠️ GitOps 工作流

### 1. Kustomize 配置模式

本项目采用 **Base + Overlays** 模式：

- **Base**: 环境无关的基础配置
  - `namespace.yaml` - 命名空间定义
  - `deployment.yaml` - 核心部署配置
  - `service.yaml` - 服务定义
  - `pvc.yaml` - 持久化存储声明
  - `kustomization.yaml` - 资源清单

- **Overlays**: 环境特定的覆盖配置
  - `prod/` - 生产环境配置
    - `ingress.yaml` - 域名和 TLS 配置
    - `kustomization.yaml` - 环境资源清单
    - 可选的补丁文件

### 2. ArgoCD 应用管理

每个应用都有对应的 ArgoCD Application 定义：

```yaml
# 示例：argocd/apps/uptime-kuma-prod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: uptime-kuma-prod
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/liuc-c/k3s-cluster-argocd.git
    path: apps/uptime-kuma/overlays/prod
    targetRevision: HEAD
  destination:
    name: in-cluster
    namespace: uptime-kuma
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 📋 部署指南

> [!WARNING]
> **配置修改提醒**
>
> 在部署任何应用之前，请务必：
> 1. 修改所有 `ingress.yaml` 文件中的域名配置
> 2. 更新 ArgoCD 应用定义中的 Git 仓库地址
> 3. 根据您的环境调整存储、资源限制等配置
> 4. 确保 DNS 记录正确指向您的集群

### 前置条件

1. **K3s 集群** 已安装并运行
2. **ArgoCD** 已安装在 `argocd` 命名空间
3. **Traefik** 作为默认入口控制器
4. **cert-manager** 已配置 Let's Encrypt
5. **DNS** 记录指向集群

### 部署新应用

1. **创建应用配置**
   ```bash
   # 创建基础配置
   mkdir -p apps/your-app/{base,overlays/prod}

   # 创建必要的 YAML 文件
   # - apps/your-app/base/
   # - apps/your-app/overlays/prod/
   ```

2. **创建 ArgoCD 应用**
   ```bash
   # 创建应用定义
   cp argocd/apps/uptime-kuma-prod-app.yaml argocd/apps/your-app-prod-app.yaml
   # 修改相应的配置
   ```

3. **应用到集群**
   ```bash
   kubectl apply -f argocd/apps/your-app-prod-app.yaml
   ```

### 启用已禁用的应用

```bash
# 启用 vaultwarden
mv argocd/apps/vaultwarden-prod-app.yaml.disable argocd/apps/vaultwarden-prod-app.yaml
kubectl apply -f argocd/apps/vaultwarden-prod-app.yaml

# 启用监控堆栈
mv argocd/apps/monitoring-prod-app.yaml.disable argocd/apps/monitoring-prod-app.yaml
kubectl apply -f argocd/apps/monitoring-prod-app.yaml
```

## 🔧 管理和维护

### ArgoCD Web UI

访问 ArgoCD 管理界面来监控和管理应用：

- **URL**: 根据您的 ArgoCD 安装配置
- **功能**:
  - 查看应用同步状态
  - 手动触发同步
  - 查看资源拓扑
  - 查看同步历史和日志

### 常用命令

```bash
# 查看 ArgoCD 应用状态
kubectl get applications -n argocd

# 查看特定应用的详细信息
kubectl describe application uptime-kuma-prod -n argocd

# 手动同步应用
kubectl patch application uptime-kuma-prod -n argocd --type merge -p '{"operation":{"sync":{}}}'

# 查看应用 Pod 状态
kubectl get pods -n uptime-kuma

# 查看应用日志
kubectl logs -f deployment/uptime-kuma -n uptime-kuma
```

### 故障排除

1. **应用同步失败**
   - 检查 Git 仓库连接
   - 验证 YAML 语法
   - 查看 ArgoCD 应用事件

2. **Ingress 无法访问**
   - 检查 DNS 解析
   - 验证 TLS 证书状态
   - 确认 Traefik 配置

3. **持久化存储问题**
   - 检查 StorageClass 配置
   - 验证 PVC 绑定状态
   - 查看存储节点状态

## 🔒 安全考虑

- **TLS 加密**: 所有外部访问都通过 HTTPS
- **证书管理**: 自动化 Let's Encrypt 证书续期
- **网络隔离**: 应用部署在独立命名空间
- **资源限制**: 设置 CPU 和内存限制
- **访问控制**: 基于 Kubernetes RBAC

## ⚙️ 配置修改指南

在使用本项目前，您需要根据自己的环境修改以下配置：

### 1. 域名配置

修改所有 `overlays/prod/ingress.yaml` 文件中的域名：

```yaml
# 示例：apps/uptime-kuma/overlays/prod/ingress.yaml
spec:
  rules:
    - host: "your-domain.com"  # 修改为您的域名
  tls:
    - hosts:
        - "your-domain.com"    # 修改为您的域名
```

### 2. Git 仓库地址

修改所有 ArgoCD 应用定义中的仓库地址：

```yaml
# 示例：argocd/apps/uptime-kuma-prod-app.yaml
spec:
  source:
    repoURL: https://github.com/YOUR-USERNAME/YOUR-REPO.git  # 修改为您的仓库
```

### 3. 证书颁发者

根据您的 cert-manager 配置修改证书颁发者：

```yaml
# 在 ingress.yaml 中
annotations:
  cert-manager.io/cluster-issuer: "your-cluster-issuer"  # 修改为您的颁发者
```

### 4. 存储和资源配置

根据您的集群资源调整：

- PVC 存储大小
- Deployment 资源限制
- 副本数量

## 📚 参考资料

- [ArgoCD 官方文档](https://argo-cd.readthedocs.io/)
- [Kustomize 官方文档](https://kustomize.io/)
- [K3s 官方文档](https://docs.k3s.io/)
- [Traefik 官方文档](https://doc.traefik.io/traefik/)

## 🤝 贡献

欢迎提交 Issue 和 Pull Request 来改进这个项目！

---

> [!CAUTION]
> **免责声明**
>
> 本项目为个人学习和实验用途，配置仅供参考。使用者需要：
>
> - 🔍 **仔细审查**所有配置文件
> - 🛠️ **根据环境调整**域名、存储、网络等配置
> - 🧪 **先在测试环境验证**再部署到生产环境
> - 📋 **备份重要数据**，作者不承担数据丢失责任
>
> **使用本项目即表示您理解并接受上述条款。**
