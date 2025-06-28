# HashiCorp Vault 和 ArgoCD Vault Plugin 部署指南

## 概述

本文档详细介绍了在 K3s 集群中部署 HashiCorp Vault 和 ArgoCD Vault Plugin 的完整流程。该部署方案遵循 GitOps 最佳实践，使用 Monorepo + ArgoCD ApplicationSet + Kustomize 模式。

## 🏗️ 架构设计

### 组件部署策略

1. **HashiCorp Vault**
   - **命名空间**: `vault`（独立命名空间，确保安全隔离）
   - **域名**: `vault.liuovo.com`
   - **存储**: 5GB 持久化存储
   - **版本**: hashicorp/vault:1.20.0

2. **ArgoCD Vault Plugin**
   - **命名空间**: `argocd`（集成到现有 ArgoCD 中）
   - **版本**: v1.18.1
   - **认证方式**: Kubernetes Service Account

### 安全考虑

- **网络隔离**: Vault 部署在独立命名空间
- **TLS 加密**: 所有外部访问通过 HTTPS
- **RBAC 权限**: 最小权限原则
- **密钥管理**: 自动化初始化和解封
- **备份策略**: 每日自动备份

## 📋 前置条件

在开始部署之前，请确保以下条件已满足：

1. **K3s 集群** 已安装并运行
2. **ArgoCD** 已安装在 `argocd` 命名空间
3. **Traefik** 作为默认入口控制器
4. **cert-manager** 已配置 Let's Encrypt
5. **DNS** 记录 `vault.liuovo.com` 指向集群
6. **kubectl** 已配置并可访问集群

## 🚀 部署步骤

### 第一步：验证 ApplicationSet 配置

确认 ApplicationSet 已正确配置并运行：

```bash
# 检查 ApplicationSet 状态
kubectl get applicationset -n argocd

# 查看 ApplicationSet 日志
kubectl logs -n argocd deployment/argocd-applicationset-controller
```

### 第二步：部署应用

由于我们使用了启用标记文件机制，应用会自动被 ApplicationSet 发现并部署：

```bash
# 检查启用的应用
find apps/ -name ".enabled-*" -type f

# 应该看到：
# apps/vault/.enabled-prod
# apps/argocd-vault-plugin/.enabled-prod
```

### 第三步：监控部署进度

```bash
# 查看 ArgoCD 中的应用状态
kubectl get applications -n argocd

# 查看 Vault 命名空间中的资源
kubectl get all -n vault

# 查看 Vault Pod 日志
kubectl logs -n vault deployment/vault
```

### 第四步：初始化 Vault

等待 Vault Pod 运行后，初始化 Job 会自动执行：

```bash
# 检查初始化 Job 状态
kubectl get jobs -n vault

# 查看初始化 Job 日志
kubectl logs -n vault job/vault-init
```

### 第五步：验证 Vault 状态

```bash
# 检查 Vault 状态
kubectl exec -n vault deployment/vault -- vault status

# 应该看到：
# Sealed: false
# Initialized: true
```

### 第六步：配置 ArgoCD Vault Plugin

ArgoCD Vault Plugin 的配置会自动应用，但需要重启 ArgoCD Repo Server：

```bash
# 重启 ArgoCD Repo Server 以加载 Vault Plugin
kubectl rollout restart deployment/argocd-repo-server -n argocd

# 等待重启完成
kubectl rollout status deployment/argocd-repo-server -n argocd
```

### 第七步：验证集成

```bash
# 检查 ArgoCD Repo Server 是否包含 Vault Plugin
kubectl exec -n argocd deployment/argocd-repo-server -- argocd-vault-plugin version

# 检查 Vault Kubernetes 认证配置
kubectl logs -n vault job/vault-k8s-auth-config
```

## 🔧 配置验证

### 验证 Vault Web UI

1. 访问 `https://vault.liuovo.com`
2. 使用根 token 登录（从 Secret 中获取）：

```bash
# 获取根 token
kubectl get secret vault-root-token -n vault -o jsonpath='{.data.token}' | base64 -d
```

### 验证 ArgoCD 集成

1. 在 ArgoCD 中创建一个测试应用
2. 在应用的 YAML 中使用 Vault 占位符：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  password: <path:secret/data/myapp#password | base64encode>
```

3. 配置应用使用 `argocd-vault-plugin`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    plugin:
      name: argocd-vault-plugin
```

## 🔐 密钥管理

### 重要密钥位置

1. **根 Token**: `vault-root-token` Secret in `vault` namespace
2. **解封密钥**: `vault-unseal-keys` Secret in `vault` namespace

### 密钥轮换

```bash
# 轮换根 token（建议定期执行）
kubectl exec -n vault deployment/vault -- vault auth -method=userpass username=admin

# 创建新的管理员 token
kubectl exec -n vault deployment/vault -- vault token create -policy=admin-policy
```

### 备份验证

```bash
# 检查备份 CronJob
kubectl get cronjobs -n vault

# 查看最近的备份
kubectl exec -n vault deployment/vault -- ls -la /backup/
```

## 🛠️ 故障排除

### Vault 无法启动

1. **检查 PVC 状态**：
```bash
kubectl get pvc -n vault
kubectl describe pvc vault-data-pvc -n vault
```

2. **检查配置**：
```bash
kubectl get configmap vault-config -n vault -o yaml
```

3. **查看详细日志**：
```bash
kubectl logs -n vault deployment/vault --previous
```

### Vault 处于密封状态

1. **手动解封**：
```bash
# 获取解封密钥
kubectl get secret vault-unseal-keys -n vault -o yaml

# 手动解封
kubectl exec -n vault deployment/vault -- vault operator unseal <key1>
kubectl exec -n vault deployment/vault -- vault operator unseal <key2>
kubectl exec -n vault deployment/vault -- vault operator unseal <key3>
```

2. **运行解封 Job**：
```bash
kubectl delete job vault-unseal -n vault
kubectl apply -f apps/vault/base/vault-unseal-job.yaml
```

### ArgoCD Vault Plugin 不工作

1. **检查 Plugin 安装**：
```bash
kubectl exec -n argocd deployment/argocd-repo-server -- which argocd-vault-plugin
kubectl exec -n argocd deployment/argocd-repo-server -- argocd-vault-plugin version
```

2. **检查环境变量**：
```bash
kubectl exec -n argocd deployment/argocd-repo-server -- env | grep VAULT
```

3. **检查 Kubernetes 认证**：
```bash
kubectl exec -n vault deployment/vault -- vault auth list
kubectl exec -n vault deployment/vault -- vault read auth/kubernetes/role/argocd
```

### 网络连接问题

1. **测试 Vault 服务连接**：
```bash
kubectl run test-pod --image=curlimages/curl --rm -it -- sh
# 在 Pod 中执行：
curl -k http://vault.vault.svc.cluster.local:8200/v1/sys/health
```

2. **检查 DNS 解析**：
```bash
kubectl run test-pod --image=busybox --rm -it -- nslookup vault.vault.svc.cluster.local
```

## 📊 监控和维护

### 健康检查

```bash
# Vault 健康状态
kubectl exec -n vault deployment/vault -- vault status

# ArgoCD 应用状态
kubectl get applications -n argocd | grep vault
```

### 日志监控

```bash
# Vault 日志
kubectl logs -n vault deployment/vault -f

# ArgoCD Repo Server 日志
kubectl logs -n argocd deployment/argocd-repo-server -f
```

### 性能监控

```bash
# 资源使用情况
kubectl top pods -n vault
kubectl top pods -n argocd
```

## 🔄 升级指南

### 升级 Vault

1. 更新镜像版本在 `apps/vault/base/deployment.yaml`
2. 提交更改到 Git
3. ArgoCD 会自动同步更新

**重要**: Vault 1.20.0 引入了重大变更：
- `disable_mlock` 现在是使用集成存储的集群的**必需配置**
- 我们的配置已经包含了此设置，因此升级是安全的
- Rekey 取消操作现在需要 nonce（10分钟内）

### 升级 ArgoCD Vault Plugin

1. 更新版本在 `apps/argocd-vault-plugin/base/argocd-repo-server-patch.yaml`
2. 提交更改到 Git
3. 重启 ArgoCD Repo Server

**注意**: 从 v1.18.0 开始，ArgoCD Vault Plugin 支持 Azure Workload Identity 和改进的 IBM Secrets Manager 集成。

## 📚 参考资料

- [HashiCorp Vault 官方文档](https://www.vaultproject.io/docs)
- [ArgoCD Vault Plugin 文档](https://argocd-vault-plugin.readthedocs.io/)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [ArgoCD 官方文档](https://argo-cd.readthedocs.io/)

## ⚠️ 安全提醒

1. **定期轮换密钥**：建议每 90 天轮换一次根 token
2. **监控访问日志**：定期检查 Vault 审计日志
3. **备份验证**：定期验证备份的完整性
4. **权限审查**：定期审查 RBAC 权限配置
5. **网络安全**：确保只有必要的服务可以访问 Vault

## 🚀 快速开始

如果您只想快速部署并测试，可以按照以下简化步骤：

```bash
# 1. 确认启用标记文件存在
ls apps/vault/.enabled-prod
ls apps/argocd-vault-plugin/.enabled-prod

# 2. 等待 ArgoCD 自动部署（约 3-5 分钟）
kubectl get applications -n argocd | grep vault

# 3. 检查 Vault 状态
kubectl get pods -n vault
kubectl exec -n vault deployment/vault -- vault status

# 4. 获取根 token
kubectl get secret vault-root-token -n vault -o jsonpath='{.data.token}' | base64 -d

# 5. 访问 Web UI
echo "访问 https://vault.liuovo.com"
```

## 📝 使用示例

### 在 ArgoCD 应用中使用 Vault 密钥

1. **在 Vault 中存储密钥**：
```bash
# 登录 Vault
export VAULT_TOKEN=$(kubectl get secret vault-root-token -n vault -o jsonpath='{.data.token}' | base64 -d)
kubectl port-forward -n vault svc/vault 8200:8200 &

# 存储密钥
vault kv put secret/myapp username=admin password=secret123
```

2. **在应用 YAML 中引用密钥**：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  username: <path:secret/data/myapp#username | base64encode>
  password: <path:secret/data/myapp#password | base64encode>
```

3. **配置 ArgoCD 应用使用 Vault Plugin**：
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    plugin:
      name: argocd-vault-plugin
```

---

**注意**: 本部署方案仅适用于个人学习和测试环境。生产环境部署请参考 HashiCorp 官方的高可用部署指南。
