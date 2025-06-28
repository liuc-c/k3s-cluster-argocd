# ApplicationSet 精确应用控制方案 - Selector 过滤器实现

## 概述

本文档描述了使用 **Selector 过滤器** 在 ArgoCD ApplicationSet 中实现精确的应用生成控制，确保只有明确启用的应用才会在 ArgoCD 中生成，未启用的应用完全不会出现在 ArgoCD UI 中。

## 问题背景

**原始问题**：当前的 ApplicationSet 会为所有发现的应用目录生成 ArgoCD Application，即使在 `app-config.yaml` 中设置了 `enabled: false`，这些应用仍然会在 ArgoCD UI 中显示。

**期望行为**：只有明确启用的应用才应该在 ArgoCD 中生成 Application 资源，未启用的应用应该完全不出现在 ArgoCD UI 中。

## 技术方案：启用标记文件 + Selector 过滤器

### 方案选择

经过深入分析 ApplicationSet 的技术限制，我们采用以下创新方案：

1. **启用标记文件约定**：使用 `.enabled-{environment}` 文件标记启用的应用环境
2. **Git 文件生成器**：发现启用标记文件而不是目录结构
3. **Selector 过滤器**：基于文件存在性进行精确过滤
4. **管理脚本**：提供便捷的启用/禁用管理工具

### 实现原理

```yaml
# Git 文件生成器发现启用标记文件
generators:
- git:
    repoURL: https://github.com/liuc-c/k3s-cluster-argocd.git
    revision: HEAD
    files:
    # 只发现启用标记文件
    - path: "apps/*/.enabled-*"
```

### 文件约定

- **启用标记文件**：`apps/{app-name}/.enabled-{environment}`
- **文件内容**：简单的 YAML 格式，包含应用和环境信息
- **控制机制**：文件存在 = 启用，文件不存在 = 禁用

## 控制机制

### 1. 启用标记文件结构

```
apps/
├── uptime-kuma/
│   ├── .enabled-prod          # 启用生产环境
│   ├── base/
│   └── overlays/
│       └── prod/
├── vaultwarden/
│   ├── .enabled-dev           # 启用开发环境
│   ├── base/
│   └── overlays/
│       └── dev/
└── monitoring/
    ├── base/
    └── overlays/
        └── prod/              # 无启用标记文件 = 禁用
```

### 2. 启用标记文件格式

```yaml
# apps/uptime-kuma/.enabled-prod
# 启用标记文件：uptime-kuma 生产环境
# 此文件的存在表示 uptime-kuma 应用的生产环境已启用
# 删除此文件将禁用该应用在 ArgoCD 中的生成

# 应用信息
app: uptime-kuma
environment: prod
enabled: true
```

### 3. 生成的应用标签

```yaml
labels:
  app.kubernetes.io/name: "uptime-kuma"
  app.kubernetes.io/environment: "prod"
  gitops.argoproj.io/enabled: "true"
  gitops.argoproj.io/controlled-by: "selector-filter"
```

## 使用指南

### 方法一：使用管理脚本（推荐）

我们提供了 PowerShell 脚本来简化启用标记文件的管理：

```powershell
# 启用应用的特定环境
.\scripts\manage-app-enablement.ps1 -Action enable -AppName uptime-kuma -Environment prod

# 禁用应用的特定环境
.\scripts\manage-app-enablement.ps1 -Action disable -AppName vaultwarden -Environment dev

# 查看特定应用的状态
.\scripts\manage-app-enablement.ps1 -Action status -AppName uptime-kuma

# 列出所有应用的启用状态
.\scripts\manage-app-enablement.ps1 -Action list
```

### 方法二：手动管理文件

#### 启用应用环境

1. **创建启用标记文件**：
   ```bash
   # 启用 uptime-kuma 的生产环境
   touch apps/uptime-kuma/.enabled-prod

   # 启用 vaultwarden 的开发环境
   touch apps/vaultwarden/.enabled-dev
   ```

2. **添加文件内容**（可选但推荐）：
   ```yaml
   # apps/uptime-kuma/.enabled-prod
   app: uptime-kuma
   environment: prod
   enabled: true
   ```

#### 禁用应用环境

1. **删除启用标记文件**：
   ```bash
   # 禁用 uptime-kuma 的生产环境
   rm apps/uptime-kuma/.enabled-prod

   # 禁用 vaultwarden 的开发环境
   rm apps/vaultwarden/.enabled-dev
   ```

### ArgoCD UI 体验

1. **精确控制**：只有有启用标记文件的应用才会在 ArgoCD 中显示
2. **整洁界面**：未启用的应用完全不会出现在 UI 中
3. **即时生效**：创建/删除启用标记文件后，ApplicationSet 会在 3 分钟内重新扫描

## 最佳实践

### 1. 文件管理

- **使用管理脚本**：推荐使用提供的 PowerShell 脚本进行统一管理
- **文件命名约定**：严格遵循 `.enabled-{environment}` 命名格式
- **版本控制**：启用标记文件应该提交到 Git 仓库中

### 2. 环境策略

- **生产环境**：谨慎启用，确保应用已经过充分测试
- **开发环境**：可以灵活启用/禁用，便于开发调试
- **测试环境**：用于验证应用配置和部署流程

### 3. 团队协作

- **清晰的文档**：在启用标记文件中添加创建时间和原因
- **代码审查**：启用/禁用操作应该通过 Pull Request 进行
- **通知机制**：重要应用的启用/禁用应该通知相关团队

### 4. 监控和维护

- **定期审查**：定期检查启用的应用列表，清理不再需要的应用
- **状态监控**：监控 ArgoCD 中应用的同步状态
- **备份策略**：保留启用标记文件的历史记录

## 故障排除

### 应用未出现在 ArgoCD 中

1. **检查启用标记文件**：
   ```bash
   # 检查文件是否存在
   ls -la apps/your-app/.enabled-*

   # 检查文件命名是否正确
   find apps/ -name ".enabled-*" -type f
   ```

2. **检查 ApplicationSet 状态**：
   ```bash
   kubectl describe applicationset config-driven-apps -n argocd
   ```

3. **检查 ApplicationSet 日志**：
   ```bash
   kubectl logs -n argocd deployment/argocd-applicationset-controller
   ```

### 应用路径错误

1. **验证目录结构**：
   ```bash
   # 确保应用目录结构正确
   tree apps/your-app/
   ```

2. **检查生成的应用配置**：
   ```bash
   kubectl get application your-app-prod -n argocd -o yaml
   ```

### 管理脚本问题

1. **检查 PowerShell 执行策略**：
   ```powershell
   Get-ExecutionPolicy
   Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
   ```

2. **验证脚本路径**：
   ```powershell
   Test-Path .\scripts\manage-app-enablement.ps1
   ```

## 技术优势

相比之前的方案，这个 Selector 过滤器方案具有以下优势：

1. **真正的精确控制**：未启用的应用完全不会在 ArgoCD 中生成
2. **简洁的实现**：不需要复杂的 Go 模板条件逻辑
3. **易于管理**：通过简单的文件操作控制应用状态
4. **清晰的界面**：ArgoCD UI 只显示启用的应用，保持整洁
5. **快速响应**：启用/禁用操作在 3 分钟内生效

## 总结

这个基于启用标记文件的 Selector 过滤器方案完美解决了 ApplicationSet 精确控制的需求：

- ✅ **完全精确控制**：只有启用的应用才会在 ArgoCD 中生成
- ✅ **整洁的 UI**：未启用的应用完全不会显示
- ✅ **简单易用**：通过文件操作或脚本轻松管理
- ✅ **纯 GitOps**：所有控制通过 Git 仓库进行
- ✅ **团队友好**：提供管理脚本和清晰的文档

这是目前在 ApplicationSet 技术限制下能够实现的最佳精确控制方案。
