# Vault 快速参考指南

## 🔑 常用命令

### Vault 管理

```bash
# 检查 Vault 状态
kubectl exec -n vault deployment/vault -- vault status

# 获取根 token
kubectl get secret vault-root-token -n vault -o jsonpath='{.data.token}' | base64 -d

# 手动解封 Vault
kubectl exec -n vault deployment/vault -- vault operator unseal <key>

# 查看 Vault 日志
kubectl logs -n vault deployment/vault -f
```

### 密钥操作

```bash
# 设置环境变量
export VAULT_TOKEN=$(kubectl get secret vault-root-token -n vault -o jsonpath='{.data.token}' | base64 -d)
kubectl port-forward -n vault svc/vault 8200:8200 &

# 存储密钥
vault kv put secret/myapp username=admin password=secret123

# 读取密钥
vault kv get secret/myapp

# 列出密钥
vault kv list secret/
```

### ArgoCD 集成

```bash
# 检查 Vault Plugin 状态
kubectl exec -n argocd deployment/argocd-repo-server -- argocd-vault-plugin version

# 重启 ArgoCD Repo Server
kubectl rollout restart deployment/argocd-repo-server -n argocd

# 查看 ArgoCD 应用状态
kubectl get applications -n argocd | grep vault
```

## 🛠️ 故障排除速查

| 问题 | 解决方案 |
|------|----------|
| Vault Pod 无法启动 | 检查 PVC 状态：`kubectl get pvc -n vault` |
| Vault 处于密封状态 | 运行解封 Job：`kubectl apply -f apps/vault/base/vault-unseal-job.yaml` |
| ArgoCD Plugin 不工作 | 重启 Repo Server：`kubectl rollout restart deployment/argocd-repo-server -n argocd` |
| 无法访问 Web UI | 检查 Ingress：`kubectl get ingress -n vault` |
| 认证失败 | 检查 K8s Auth：`kubectl logs -n vault job/vault-k8s-auth-config` |

## 📊 监控检查点

```bash
# 健康检查
kubectl get pods -n vault
kubectl get pods -n argocd | grep repo-server

# 资源使用
kubectl top pods -n vault
kubectl top pods -n argocd

# 备份状态
kubectl get cronjobs -n vault
kubectl get pvc vault-backup-pvc -n vault
```

## 🔐 安全检查清单

- [ ] Vault 已初始化并解封
- [ ] 根 token 已安全存储
- [ ] 解封密钥已备份
- [ ] TLS 证书有效
- [ ] RBAC 权限正确配置
- [ ] 备份 CronJob 正常运行
- [ ] 审计日志已启用（可选）

## 📱 Web UI 访问

1. **URL**: https://vault.liuovo.com
2. **登录方式**: Token
3. **Token 获取**:
   ```bash
   kubectl get secret vault-root-token -n vault -o jsonpath='{.data.token}' | base64 -d
   ```

## 🔄 维护任务

### 每日
- 检查 Vault 状态
- 验证备份完成

### 每周
- 检查资源使用情况
- 审查访问日志

### 每月
- 轮换访问 token
- 更新密钥策略
- 验证备份恢复

### 每季度
- 轮换根 token
- 安全审计
- 更新文档
