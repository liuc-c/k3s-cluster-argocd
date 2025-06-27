# Argo CD 核心配置变更说明

## 目的

此目录下的 `argocd-cm-patch.yaml` 文件是一个关键的配置文件补丁。

它的主要作用是修改 Argo CD 的核心配置 `argocd-cm` (ConfigMap)，**为 Argo CD 中所有使用 Kustomize 的应用，全局开启 Helm 支持。**

如果不应用此补丁，任何尝试使用 Kustomize 来渲染 Helm Chart 的 Argo CD 应用（例如 `monitoring-prod` 应用）都将构建失败。

## ⚠️ 重要提示：必须手动操作

对 `argocd-cm` 的修改，相当于在修改 Argo CD 自身的“大脑”。这是一个敏感操作，Argo CD 为了自身稳定，禁止通过常规的 GitOps 自动同步流程来修改自己。

因此，**此项变更必须由管理员在终端中手动执行。** 它不能，也不应该被包含在 `app-of-apps` 的自动部署模式中。

---

## 操作步骤

请按照以下两个步骤来应用配置并使其生效。

### 第 1 步：应用配置补丁

使用 `kubectl apply` 命令来将补丁应用到集群中。这个命令是安全的，可以重复执行。

```bash
# 确保您的 kubectl 上下文已正确指向目标集群

# 应用补丁文件
# (请将 "argocd" 替换为您的 Argo CD 实际安装的命名空间)
kubectl apply -f argocd-cm-patch.yaml -n argocd
```

### 第 2 步：重启 Repo Server (关键步骤)

`repo-server` 是 Argo CD 中负责拉取 Git 仓库、渲染 YAML 的组件。我们必须重启它，它才会加载刚才应用的新配置。

```bash
# 重启 repo-server deployment 来加载新的配置
# (请将 "argocd" 替换为您的 Argo CD 实际安装的命名空间)
kubectl rollout restart deployment argocd-repo-server -n argocd
```

---

## 如何验证

操作完成后，您可以通过以下任一方式确认配置已生效：

1.  **检查 ConfigMap 内容：**
    运行以下命令，检查输出的 `data` 字段中是否包含 `kustomize.buildOptions: --enable-helm`。
    ```bash
    kubectl get configmap argocd-cm -n argocd -o yaml
    ```

2.  **在 Argo CD UI 中验证：**
    *   进入 Argo CD 的 Web UI。
    *   找到之前因 Kustomize 无法渲染 Helm 而失败的应用。
    *   点击应用的 **Refresh** 按钮，然后尝试 **Sync**。如果应用能够成功同步，说明配置已生效。
