# CAPBM v1beta2 迁移与重新部署指南

## 概述

CAPBM API 已从 `infrastructure.cluster.x-k8s.io/v1beta1` 迁移到 `infrastructure.cluster.x-k8s.io/v1beta2`。

### 变更内容

| 变更前 (v1beta1) | 变更后 (v1beta2) |
|-----------------|-----------------|
| `api/v1beta1/` 目录 | `api/v1beta2/` 目录 |
| `GroupVersion.Version = "v1beta1"` | `GroupVersion.Version = "v1beta2"` |
| CRD 标签: `cluster.x-k8s.io/v1beta1=v1beta1` | CRD 标签: `cluster.x-k8s.io/v1beta2=v1beta2` |
| ClusterClass 引用: `infrastructure.cluster.x-k8s.io/v1beta1` | ClusterClass 引用: `infrastructure.cluster.x-k8s.io/v1beta2` |

### 字段结构

**v1beta2 的字段结构与 v1beta1 完全相同**，无需修改现有资源配置。

---

## 重新部署流程

### 步骤 1：备份现有资源

```bash
# 备份现有 CRD
kubectl get crd -l cluster.x-k8s.io/provider=infrastructure-baremetal -o yaml > crd-backup.yaml

# 备份现有 ClusterClass 和模板
kubectl get clusterclass,baremetalclustertemplate,baremetalmachinetemplate,kubeadmcontrolplanetemplate,kubeadmconfigtemplate -A -o yaml > clusterclass-backup.yaml

# 备份现有 Cluster 资源
kubectl get cluster -A -o yaml > cluster-backup.yaml
```

### 步骤 2：删除现有 Cluster 资源

```bash
# 删除所有 Cluster 资源（不会删除工作负载集群）
kubectl delete cluster --all -A

# 等待资源完全删除
kubectl get cluster -A
```

### 步骤 3：删除现有 ClusterClass 和模板

```bash
# 删除 ClusterClass
kubectl delete clusterclass baremetal-clusterclass -n default

# 删除模板资源
kubectl delete baremetalclustertemplate baremetal-cluster-template -n default
kubectl delete baremetalmachinetemplate baremetal-control-plane-machine -n default
kubectl delete baremetalmachinetemplate baremetal-worker-machine -n default
kubectl delete kubeadmcontrolplanetemplate baremetal-kubeadm-control-plane -n default
kubectl delete kubeadmconfigtemplate baremetal-kubeadm-worker -n default
```

### 步骤 4：删除现有 CRD

```bash
# 删除旧版 CRD
kubectl delete crd baremetalclusters.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalclustertemplates.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalmachines.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalmachinetemplates.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalhostinventories.infrastructure.cluster.x-k8s.io
```

### 步骤 5：重新部署 CAPBM Controller

```bash
# 拉取最新代码
git pull origin main

# 构建并部署
kubectl apply -k modules/capbm/config/crd/
kubectl apply -k modules/capbm/config/

# 验证 Controller 运行状态
kubectl get pods -n capbm-system
```

### 步骤 6：验证新 CRD

```bash
# 验证 CRD 版本
kubectl get crd baremetalclusters.infrastructure.cluster.x-k8s.io -o jsonpath='{.spec.versions[*].name}'
# 预期输出: v1beta2

# 验证 CRD 标签
kubectl get crd baremetalclusters.infrastructure.cluster.x-k8s.io -o jsonpath='{.metadata.labels}'
# 预期输出应包含: cluster.x-k8s.io/v1beta2=v1beta2
```

### 步骤 7：重新部署 ClusterClass

```bash
# 部署新版 ClusterClass（使用 v1beta2 API 版本）
kubectl apply -k modules/capbm/config/clusterclass/

# 验证 ClusterClass 部署
kubectl get clusterclass baremetal-clusterclass -n default
kubectl get baremetalclustertemplate,baremetalmachinetemplate,kubeadmcontrolplanetemplate,kubeadmconfigtemplate -n default
```

### 步骤 8：重新创建 Cluster

```bash
# 使用新模板创建 Cluster
kubectl apply -f cluster.yaml

# 验证 Cluster 状态
kubectl describe cluster single-node-cluster -n default
```

---

## 验证清单

- [ ] CRD 版本为 `v1beta2`
- [ ] CRD 标签包含 `cluster.x-k8s.io/v1beta2=v1beta2`
- [ ] CAPBM Controller 运行正常
- [ ] ClusterClass 部署成功
- [ ] 所有模板资源使用 `v1beta2` API 版本
- [ ] Cluster 资源创建成功
- [ ] 工作负载集群正常创建

---

## 常见问题

### Q1: 删除 CRD 时提示 "resource in use"

**原因**：仍有资源引用该 CRD。

**解决方案**：
```bash
# 检查是否有残留资源
kubectl get baremetalcluster,baremetalmachine,baremetalhostinventory -A

# 删除残留资源
kubectl delete baremetalcluster,baremetalmachine,baremetalhostinventory --all -A
```

### Q2: ClusterClass 部署失败

**原因**：模板资源 API 版本不匹配。

**解决方案**：确保所有模板资源使用 `v1beta2` API 版本：
```bash
kubectl get baremetalclustertemplate -o jsonpath='{.items[*].apiVersion}'
# 预期输出: infrastructure.cluster.x-k8s.io/v1beta2
```

### Q3: 工作负载集群是否需要重建？

**不需要**。工作负载集群由 kubeadm 管理，与 CAPBM API 版本无关。只需重新创建管理集群中的 Cluster 资源即可。

---

## 回滚方案

如果需要回滚到 v1beta1：

```bash
# 1. 切换到 v1beta1 分支或 tag
git checkout <v1beta1-tag>

# 2. 删除 v1beta2 CRD
kubectl delete crd baremetalclusters.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalclustertemplates.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalmachines.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalmachinetemplates.infrastructure.cluster.x-k8s.io
kubectl delete crd baremetalhostinventories.infrastructure.cluster.x-k8s.io

# 3. 重新部署 v1beta1 CRD 和 Controller
kubectl apply -k modules/capbm/config/crd/
kubectl apply -k modules/capbm/config/

# 4. 重新部署 ClusterClass 和 Cluster
kubectl apply -k modules/capbm/config/clusterclass/
kubectl apply -f cluster.yaml
```
