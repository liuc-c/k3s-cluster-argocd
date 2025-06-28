# ApplicationSet Selector 过滤器方案

## 🎯 方案概述

这是一个基于 **启用标记文件** 和 **Selector 过滤器** 的 ApplicationSet 精确控制方案，实现了真正的精确应用生成控制。

### 核心特性

- ✅ **完全精确控制**：只有启用的应用才会在 ArgoCD 中生成
- ✅ **整洁的 UI**：未启用的应用完全不会显示在 ArgoCD 中
- ✅ **简单易用**：通过文件操作或管理脚本轻松控制
- ✅ **纯 GitOps**：所有控制通过 Git 仓库进行
- ✅ **即时生效**：启用/禁用操作在 3 分钟内生效

## 🔧 工作原理

### 1. 启用标记文件约定

```
apps/
├── uptime-kuma/
│   ├── .enabled-prod          # 启用生产环境
│   ├── base/
│   └── overlays/prod/
├── vaultwarden/
│   ├── .enabled-dev           # 启用开发环境
│   ├── base/
│   └── overlays/dev/
└── monitoring/
    └── overlays/prod/         # 无标记文件 = 禁用
```

### 2. ApplicationSet 配置

```yaml
# 使用 Git 文件生成器发现启用标记文件
generators:
- git:
    repoURL: https://github.com/liuc-c/k3s-cluster-argocd.git
    revision: HEAD
    files:
    # 只发现启用标记文件
    - path: "apps/*/.enabled-*"
```

### 3. 应用生成逻辑

- **文件存在** → 应用在 ArgoCD 中生成并自动同步
- **文件不存在** → 应用完全不会在 ArgoCD 中出现

## 🚀 快速开始

### 使用管理脚本（推荐）

```powershell
# 启用应用
.\scripts\manage-app-enablement.ps1 -Action enable -AppName uptime-kuma -Environment prod

# 禁用应用
.\scripts\manage-app-enablement.ps1 -Action disable -AppName vaultwarden -Environment dev

# 查看状态
.\scripts\manage-app-enablement.ps1 -Action list
```

### 手动管理

```bash
# 启用应用
touch apps/uptime-kuma/.enabled-prod

# 禁用应用
rm apps/vaultwarden/.enabled-dev

# 查看启用状态
find apps/ -name ".enabled-*" -type f
```

## 📋 使用示例

### 启用 uptime-kuma 生产环境

```bash
# 方法1：使用脚本
.\scripts\manage-app-enablement.ps1 -Action enable -AppName uptime-kuma -Environment prod

# 方法2：手动创建文件
echo "app: uptime-kuma
environment: prod
enabled: true
```

### 禁用 vaultwarden 开发环境

```bash
# 方法1：使用脚本
.\scripts\manage-app-enablement.ps1 -Action disable -AppName vaultwarden -Environment dev

# 方法2：手动删除文件
rm apps/vaultwarden/.enabled-dev
```

## 🎨 ArgoCD UI 体验

- **精确显示**：只有启用的应用才会在 ArgoCD UI 中显示
- **整洁界面**：未启用的应用完全不会出现
- **自动同步**：启用的应用自动配置同步策略
- **即时更新**：文件变更后 3 分钟内生效

## 📚 详细文档

完整的技术文档和使用指南请参考：
- [ApplicationSet 精确控制详细文档](docs/applicationset-precise-control.md)

## 🔄 迁移指南

如果您之前使用的是基于 `app-config.yaml` 的方案，请按以下步骤迁移：

1. **备份现有配置**：
   ```bash
   cp -r apps/ apps-backup/
   ```

2. **创建启用标记文件**：
   ```bash
   # 为每个需要启用的应用环境创建标记文件
   touch apps/uptime-kuma/.enabled-prod
   touch apps/vaultwarden/.enabled-dev
   ```

3. **部署新的 ApplicationSet**：
   ```bash
   kubectl apply -f argocd/applicationset.yaml
   ```

4. **验证结果**：
   ```bash
   kubectl get applications -n argocd
   ```

## ⚠️ 注意事项

1. **文件命名**：必须严格遵循 `.enabled-{environment}` 格式
2. **Git 提交**：启用标记文件必须提交到 Git 仓库
3. **权限管理**：启用/禁用操作建议通过 Pull Request 进行
4. **环境支持**：当前支持 `prod`、`dev`、`staging` 环境

## 🛠️ 故障排除

### 应用未出现在 ArgoCD 中

1. 检查启用标记文件是否存在且已提交到 Git
2. 验证文件命名格式是否正确
3. 查看 ApplicationSet 控制器日志

### 管理脚本无法运行

1. 检查 PowerShell 执行策略
2. 确认脚本路径正确
3. 验证应用目录结构

---

这个方案完美解决了 ApplicationSet 精确控制的需求，提供了最佳的用户体验和管理便利性。
