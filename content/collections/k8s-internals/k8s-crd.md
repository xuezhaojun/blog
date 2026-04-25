---
title: "从一次 CR 删不掉聊起：彻底搞懂 CRD、Finalizer 和 Webhook"
date: 2026-04-24
draft: true
tags: ["Kubernetes", "CRD", "Operator", "Webhook", "中文"]
summary: "kubectl delete 某个 CR 后一直卡在 Terminating，--force 也无效。排查过程串联 CRD 的设计哲学：自定义资源如何在 API Server 中'活起来'、Finalizer 如何保证安全清理、Webhook 如何扩展准入控制，以及那些让人抓狂的版本迁移坑。"
weight: 3
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## 一个删不掉的 CR

你维护一个多集群管理平台，用 CRD 定义了一种叫 `ManagedCluster` 的资源，代表一个被纳管的 Kubernetes 集群。

某天，你想清理一个已经下线的集群：

```bash
kubectl delete managedcluster cluster-east-02
```

等了一分钟，没有返回。`kubectl get` 一看：

```
NAME               STATUS        AGE
cluster-east-02    Terminating   62s
```

你心想可能是卡了，试试 `--force`：

```bash
kubectl delete managedcluster cluster-east-02 --force --grace-period=0
```

还是 Terminating。五分钟、十分钟，纹丝不动。

你检查了 controller 的日志——**controller Pod 半小时前因为节点驱逐被杀了，还没恢复。**

**为什么 CR 删不掉？为什么 `--force` 也没用？** 要回答这个问题，我们需要从 CRD 的设计哲学、Finalizer 的工作原理、以及 Webhook 的准入控制机制说起。

---

## CRD 是什么

### Kubernetes 的扩展点

Kubernetes 本身自带了 Pod、Service、Deployment 这些资源类型。但如果你想管理"集群"、"数据库实例"、"证书"这些 Kubernetes 不认识的东西呢？

CRD（Custom Resource Definition）就是 Kubernetes 提供的**资源类型注册机制**。你告诉 API Server："我要定义一种新的资源类型叫 `ManagedCluster`"，API Server 就会**自动生成一套完整的 RESTful 端点**：

```
GET    /apis/multicluster.example.com/v1/managedclusters
POST   /apis/multicluster.example.com/v1/managedclusters
GET    /apis/multicluster.example.com/v1/managedclusters/{name}
PUT    /apis/multicluster.example.com/v1/managedclusters/{name}
DELETE /apis/multicluster.example.com/v1/managedclusters/{name}
```

不需要写任何 API 代码。CRD 一注册，`kubectl get managedclusters` 就能用了。

### CRD 只定义数据模型，没有业务逻辑

这是一个关键区分。CRD 只做一件事：**告诉 API Server 你的资源长什么样**——有哪些字段、什么类型、哪些是必填的。API Server 负责存储（写到 etcd）和提供 CRUD API。

但 API Server 不会帮你做任何业务逻辑。你定义了 `ManagedCluster`，API Server 不知道怎么去真正注册一个集群、配置网络、同步配置。这些"怎么做"的逻辑，需要你写一个 **Controller**。

### CRD + Controller = 完整扩展

这是 Kubernetes 扩展的核心模式：

```
CRD    → 定义"是什么"（数据模型）
Controller → 实现"做什么"（业务逻辑）
```

用户通过 `kubectl apply` 创建一个 CR（Custom Resource，CRD 的实例），Controller watch 到这个 CR，执行相应的操作，把结果写回 CR 的 `.status`。

这就是 Operator 模式：**用 Kubernetes 原生的方式管理任何东西。**

---

## CRD 定义示例

下面是一个完整的 CRD 定义，用来管理集群资源：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusters.multicluster.example.com   # 命名规则: <plural>.<group>
spec:
  group: multicluster.example.com            # API 组名
  names:
    kind: Cluster                            # 资源类型名（大驼峰）
    listKind: ClusterList                    # 列表类型名
    plural: clusters                         # URL 路径中的复数形式
    singular: cluster                        # 单数形式
    shortNames:
      - cls                                  # kubectl get cls
  scope: Cluster                             # Cluster 级别（非 Namespaced）
  versions:
    - name: v1
      served: true                           # 该版本是否通过 API 提供服务
      storage: true                          # 该版本是否用于 etcd 存储
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - kubeAPIServer
              properties:
                kubeAPIServer:
                  type: string
                  description: "集群 API Server 地址"
                provider:
                  type: string
                  enum: ["aws", "gcp", "azure", "baremetal"]
                  description: "云提供商"
                version:
                  type: string
                  pattern: "^v\\d+\\.\\d+\\.\\d+$"
                  description: "Kubernetes 版本，如 v1.28.0"
            status:
              type: object
              properties:
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                        enum: ["True", "False", "Unknown"]
                      lastTransitionTime:
                        type: string
                        format: date-time
                      reason:
                        type: string
                      message:
                        type: string
                phase:
                  type: string
                  enum: ["Pending", "Running", "Failed", "Unknown"]
      subresources:
        status: {}                           # 启用 /status 子资源
      additionalPrinterColumns:              # kubectl get 输出列
        - name: API Server
          type: string
          jsonPath: .spec.kubeAPIServer
        - name: Provider
          type: string
          jsonPath: .spec.provider
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
```

应用这个 CRD 之后，你就可以创建 CR 了：

```yaml
apiVersion: multicluster.example.com/v1
kind: Cluster
metadata:
  name: cluster-east-02
spec:
  kubeAPIServer: "https://10.0.1.100:6443"
  provider: aws
  version: v1.28.3
```

```bash
$ kubectl apply -f cluster-east-02.yaml
cluster.multicluster.example.com/cluster-east-02 created

$ kubectl get cls
NAME               API SERVER                  PROVIDER   PHASE     AGE
cluster-east-02    https://10.0.1.100:6443     aws        Pending   5s
```

注意 Phase 是 `Pending`——因为还没有 Controller 来处理它。CR 只是数据，躺在 etcd 里，等着 Controller 来赋予它生命。

---

## Custom Controller 开发流程（kubebuilder）

手写一个 Controller 涉及大量样板代码：client 初始化、informer 配置、work queue 管理、leader election……kubebuilder 帮你生成这些。

### 初始化项目

```bash
# 创建项目目录
mkdir cluster-controller && cd cluster-controller

# 初始化项目（指定域名和仓库）
kubebuilder init --domain example.com --repo github.com/example/cluster-controller

# 创建 API（CRD + Controller）
kubebuilder create api --group multicluster --version v1 --kind Cluster
```

kubebuilder 会问你是否要创建 Resource 和 Controller，都选 yes。

### 生成的目录结构

```
cluster-controller/
├── cmd/
│   └── main.go                    # 程序入口
├── api/
│   └── v1/
│       ├── cluster_types.go       # ← CRD 的 Go 结构体定义
│       ├── groupversion_info.go   # GVK 注册
│       └── zz_generated.deepcopy.go
├── internal/
│   └── controller/
│       ├── cluster_controller.go  # ← Controller 逻辑（你写代码的地方）
│       └── cluster_controller_test.go
├── config/
│   ├── crd/                       # 生成的 CRD YAML
│   ├── rbac/                      # RBAC 权限配置
│   ├── manager/                   # Deployment 配置
│   └── webhook/                   # Webhook 配置
├── Dockerfile
├── Makefile
└── PROJECT                        # kubebuilder 元数据
```

核心文件就两个：`cluster_types.go` 定义数据模型，`cluster_controller.go` 实现业务逻辑。

### Reconcile 函数

Controller 的核心是 Reconcile 函数。Kubernetes 的 Controller 模式是**声明式的**：你不是在响应事件（"资源被创建了"），而是在回答一个问题——**"当前状态和期望状态一致吗？不一致就修。"**

kubebuilder 生成的骨架：

```go
package controller

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    multiclusterv1 "github.com/example/cluster-controller/api/v1"
)

type ClusterReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=multicluster.example.com,resources=clusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=multicluster.example.com,resources=clusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=multicluster.example.com,resources=clusters/finalizers,verbs=update

func (r *ClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // 1. 获取 CR
    var cluster multiclusterv1.Cluster
    if err := r.Get(ctx, req.NamespacedName, &cluster); err != nil {
        // CR 被删除了（且没有 Finalizer），直接返回
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 检查是否在被删除
    if !cluster.DeletionTimestamp.IsZero() {
        // 处理删除逻辑（见 Finalizer 章节）
        logger.Info("Cluster is being deleted", "name", cluster.Name)
        return ctrl.Result{}, nil
    }

    // 3. 核心 Reconcile 逻辑
    logger.Info("Reconciling cluster", "name", cluster.Name)

    // 检查集群连通性
    healthy, err := r.checkClusterHealth(ctx, &cluster)
    if err != nil {
        logger.Error(err, "Failed to check cluster health")
        // 返回错误会触发指数退避重试
        return ctrl.Result{}, err
    }

    // 4. 更新 Status
    if healthy {
        cluster.Status.Phase = "Running"
    } else {
        cluster.Status.Phase = "Failed"
    }
    if err := r.Status().Update(ctx, &cluster); err != nil {
        return ctrl.Result{}, err
    }

    // 5. 定期重新检查（每 30 秒）
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *ClusterReconciler) checkClusterHealth(ctx context.Context, cluster *multiclusterv1.Cluster) (bool, error) {
    // 实际中：用 kubeAPIServer 地址连接集群，检查 /healthz
    return true, nil
}

func (r *ClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&multiclusterv1.Cluster{}).
        Complete(r)
}
```

几个关键点：

- **`ctrl.Result{}`** — 不重试，一切正常
- **`ctrl.Result{RequeueAfter: 30s}`** — 30 秒后再 reconcile 一次
- **返回 `error`** — 会自动进入指数退避重试队列
- **`client.IgnoreNotFound(err)`** — CR 已经不存在了，不需要 reconcile

---

## Finalizer 模式

### 为什么需要 Finalizer

假设你的 `Cluster` CR 对应了真实的外部资源：在云上创建了 VPC、安全组、IAM 角色。如果用户 `kubectl delete cluster cluster-east-02`，CR 从 etcd 删掉了，但那些云资源怎么办？

**没有 Finalizer 的情况：**

```
用户 delete CR → API Server 立即从 etcd 删除 → CR 消失 → 云资源成了孤儿
```

Controller 连知道 CR 被删了的机会都没有。

**有 Finalizer 的情况：**

```
用户 delete CR
→ API Server 看到有 Finalizer，不删除，只设置 deletionTimestamp
→ CR 进入 Terminating 状态
→ Controller 检测到 deletionTimestamp 不为空
→ 执行清理逻辑（删 VPC、安全组、IAM……）
→ 清理完成，移除 Finalizer
→ API Server 发现没有 Finalizer 了，真正删除 CR
```

Finalizer 本质是一个**删除保护锁**。只要 `metadata.finalizers` 列表非空，API Server **拒绝真正删除这个对象**。

### 实现代码

```go
const clusterFinalizer = "multicluster.example.com/cluster-cleanup"

func (r *ClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    var cluster multiclusterv1.Cluster
    if err := r.Get(ctx, req.NamespacedName, &cluster); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 检查是否在被删除
    if !cluster.DeletionTimestamp.IsZero() {
        // CR 正在被删除，执行清理
        if containsString(cluster.Finalizers, clusterFinalizer) {
            logger.Info("Running finalizer cleanup", "cluster", cluster.Name)

            // 清理外部资源
            if err := r.cleanupExternalResources(ctx, &cluster); err != nil {
                // 清理失败，返回错误，会自动重试
                logger.Error(err, "Failed to cleanup external resources")
                return ctrl.Result{}, err
            }

            // 清理完成，移除 Finalizer
            cluster.Finalizers = removeString(cluster.Finalizers, clusterFinalizer)
            if err := r.Update(ctx, &cluster); err != nil {
                return ctrl.Result{}, err
            }
            logger.Info("Finalizer removed, CR will be deleted", "cluster", cluster.Name)
        }
        return ctrl.Result{}, nil
    }

    // 不是删除操作，确保 Finalizer 存在
    if !containsString(cluster.Finalizers, clusterFinalizer) {
        cluster.Finalizers = append(cluster.Finalizers, clusterFinalizer)
        if err := r.Update(ctx, &cluster); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 正常 Reconcile 逻辑...
    return ctrl.Result{}, nil
}

func (r *ClusterReconciler) cleanupExternalResources(ctx context.Context, cluster *multiclusterv1.Cluster) error {
    // 删除 VPC、安全组、IAM 角色等
    // 如果任何步骤失败，返回 error，触发重试
    return nil
}

func containsString(slice []string, s string) bool {
    for _, item := range slice {
        if item == s {
            return true
        }
    }
    return false
}

func removeString(slice []string, s string) []string {
    var result []string
    for _, item := range slice {
        if item != s {
            result = append(result, item)
        }
    }
    return result
}
```

### Controller 挂了 → CR 卡在 Terminating

回到开头的事故。现在你理解了：

1. `ManagedCluster` CR 有 Finalizer
2. 用户执行了 `kubectl delete`
3. API Server 设置了 `deletionTimestamp`，CR 进入 Terminating
4. 但 Controller Pod 被驱逐了，没有运行
5. 没有人执行清理逻辑，没有人移除 Finalizer
6. API Server 看到 Finalizer 还在，拒绝删除

`--force --grace-period=0` 只影响 **Pod 的优雅终止**，对 Finalizer 完全无效。Finalizer 是 API Server 层面的机制，不是 kubelet 层面的。

**修复方式：**

方案一：恢复 Controller，让它正常执行清理和移除 Finalizer。

方案二：如果你确认外部资源已经手动清理了（或者不需要清理），直接手动移除 Finalizer：

```bash
kubectl patch managedcluster cluster-east-02 \
  --type=json \
  -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```

Finalizer 一移除，API Server 立即删除 CR。

---

## Webhook（准入控制扩展）

### 什么是 Admission Webhook

当你 `kubectl apply` 一个资源时，请求经历这样的流程：

```
kubectl → API Server → 认证(AuthN) → 授权(AuthZ) → Mutating Webhook → Validating Webhook → etcd
```

Webhook 让你在资源**写入 etcd 之前**拦截和修改请求。有两种：

**Mutating Webhook** — 修改请求内容。比如自动注入 sidecar、设置默认值、添加 label。

**Validating Webhook** — 校验请求内容。比如检查字段合法性、强制命名规范、限制资源配额。

执行顺序很重要：**先 Mutating，后 Validating**。这意味着 Validating Webhook 校验的是 Mutating Webhook 修改过的最终版本。

### kubebuilder 创建 Webhook

```bash
kubebuilder create webhook --group multicluster --version v1 --kind Cluster \
  --defaulting --validation
```

这会生成 `api/v1/cluster_webhook.go`：

```go
package v1

import (
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

var clusterlog = logf.Log.WithName("cluster-resource")

func (r *Cluster) SetupWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).
        For(r).
        Complete()
}

// +kubebuilder:webhook:path=/mutate-multicluster-example-com-v1-cluster,mutating=true,failurePolicy=fail,sideEffects=None,groups=multicluster.example.com,resources=clusters,verbs=create;update,versions=v1,name=mcluster.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &Cluster{}

// Default 实现 Mutating Webhook（设置默认值）
func (r *Cluster) Default() {
    clusterlog.Info("setting defaults", "name", r.Name)

    // 如果没指定 provider，默认 baremetal
    if r.Spec.Provider == "" {
        r.Spec.Provider = "baremetal"
    }

    // 自动添加 label
    if r.Labels == nil {
        r.Labels = make(map[string]string)
    }
    r.Labels["multicluster.example.com/managed"] = "true"
}

// +kubebuilder:webhook:path=/validate-multicluster-example-com-v1-cluster,mutating=false,failurePolicy=fail,sideEffects=None,groups=multicluster.example.com,resources=clusters,verbs=create;update;delete,versions=v1,name=vcluster.kb.io,admissionReviewVersions=v1

var _ webhook.Validator = &Cluster{}

// ValidateCreate 创建时校验
func (r *Cluster) ValidateCreate() (admission.Warnings, error) {
    clusterlog.Info("validate create", "name", r.Name)

    if r.Spec.KubeAPIServer == "" {
        return nil, fmt.Errorf("kubeAPIServer is required")
    }
    return nil, nil
}

// ValidateUpdate 更新时校验
func (r *Cluster) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
    clusterlog.Info("validate update", "name", r.Name)

    oldCluster := old.(*Cluster)
    // 不允许修改 provider（不可变字段）
    if r.Spec.Provider != oldCluster.Spec.Provider {
        return nil, fmt.Errorf("provider is immutable, cannot change from %s to %s",
            oldCluster.Spec.Provider, r.Spec.Provider)
    }
    return nil, nil
}

// ValidateDelete 删除时校验
func (r *Cluster) ValidateDelete() (admission.Warnings, error) {
    clusterlog.Info("validate delete", "name", r.Name)
    return nil, nil
}
```

### Webhook 的部署

Webhook 本质是一个 HTTPS 服务。API Server 在处理请求时，向你的 Webhook 服务发送 `AdmissionReview` 请求，Webhook 返回允许/拒绝/修改后的结果。

这意味着 Webhook 需要：

1. **TLS 证书** — API Server 只通过 HTTPS 调用 Webhook
2. **Service** — Webhook 需要一个 Kubernetes Service 提供网络可达
3. **MutatingWebhookConfiguration / ValidatingWebhookConfiguration** — 告诉 API Server 什么资源、什么操作要发给哪个 Webhook

kubebuilder + cert-manager 可以自动管理这些。但你必须理解这个链路，因为**任何一环断了，Webhook 就会阻塞所有相关请求**。

---

## 面试追问

### Q1: CRD vs API Aggregation，什么时候选哪个？

**CRD** 是轻量级方案：

- 不需要写 API Server 代码
- API Server 自动处理存储（etcd）、认证、授权、CRUD
- 数据就是 JSON/YAML，存在 kube-apiserver 的 etcd 里
- 适合大多数场景

**API Aggregation** 是重量级方案：

- 你自己写一个 API Server（实现 `apiserver` 包的接口）
- 通过 `APIService` 注册到 kube-apiserver，kube-apiserver 会把特定 API 组的请求代理给你
- 你的 API Server 可以用自己的存储后端（不限于 etcd）
- 可以实现子资源、自定义协议协商等 CRD 不支持的功能

什么时候需要 API Aggregation？

- 需要自定义存储后端（比如 metrics-server 数据在内存里，不适合 etcd）
- 需要 CRD 不支持的高级 API 特性（如 `kubectl exec` 这种 WebSocket 子资源）
- 数据量巨大，不想挤占 kube-apiserver 的 etcd
- 需要实现 protobuf 协议支持以提升性能

**大多数情况下选 CRD。** 只有当 CRD 明确满足不了需求时才考虑 API Aggregation。

### Q2: CRD 多版本怎么管理？Conversion Webhook 是什么？

真实项目中，CRD 的 schema 会演进。比如 v1alpha1 里字段叫 `serverURL`，v1beta1 里改成了 `kubeAPIServer`。

Kubernetes 要求 etcd 只存一个版本（`storage: true` 的那个），但 API 可以同时提供多个版本（`served: true`）。当用户用 v1alpha1 请求、但 etcd 存的是 v1beta1 时，就需要**版本转换**。

```yaml
versions:
  - name: v1beta1
    served: true
    storage: true    # etcd 用这个版本
  - name: v1alpha1
    served: true     # API 也提供这个版本
    storage: false
```

转换方式有两种：

1. **None 策略** — 只允许字段名完全一致的版本共存，基本没用
2. **Conversion Webhook** — 你写一个 Webhook，实现版本间的字段映射

```go
// Conversion Webhook 需要实现的逻辑（伪代码）
func convert(src *v1alpha1.Cluster, dst *v1beta1.Cluster) {
    dst.Spec.KubeAPIServer = src.Spec.ServerURL  // 字段重命名
    dst.Spec.Provider = src.Spec.Cloud            // 字段重命名
    // dst.Spec.NewField = 默认值                 // 新版本才有的字段
}
```

CRD 中配置 Conversion Webhook：

```yaml
spec:
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          namespace: system
          name: cluster-controller-webhook
          path: /convert
```

**坑：** 如果 Conversion Webhook 挂了，所有跨版本的请求都会失败。而且如果 `storedVersions` 里还有旧版本，你甚至不能删除旧版本的定义。

### Q3: Finalizer 卡住了怎么排查？

排查思路：

**第一步：确认 Finalizer 列表**

```bash
kubectl get <resource> <name> -o jsonpath='{.metadata.finalizers}'
```

**第二步：看 deletionTimestamp**

```bash
kubectl get <resource> <name> -o jsonpath='{.metadata.deletionTimestamp}'
```

如果有 `deletionTimestamp` 且有 Finalizer，说明在等 Controller 清理。

**第三步：检查 Controller 状态**

```bash
# Controller Pod 在运行吗？
kubectl get pods -n <namespace> | grep controller

# Controller 日志有没有报错？
kubectl logs -n <namespace> <controller-pod> | grep -i "finalizer\|cleanup\|error"
```

**第四步：常见原因**

| 原因 | 表现 | 解决 |
|---|---|---|
| Controller 没在运行 | Pod 不存在或 CrashLoopBackOff | 恢复 Controller |
| Controller RBAC 不够 | 日志有 `forbidden` 错误 | 补充 RBAC 权限 |
| 清理外部资源失败 | 日志有清理相关错误 | 修复外部资源问题，或手动清理后去掉 Finalizer |
| Webhook 阻塞了 Update | Controller 的 patch/update 请求被 Webhook 拒绝 | 检查 Webhook 日志 |

**最后手段：手动移除 Finalizer**

```bash
kubectl patch <resource> <name> \
  --type=json \
  -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```

> 注意：手动移除 Finalizer 意味着跳过了清理逻辑，可能留下孤儿资源。只在你确认外部资源已经不存在时才用。

### Q4: Reconcile 是串行还是并行的？

**同一个 key（namespace/name）是串行的。** Controller-runtime 使用 work queue，保证同一个对象的 reconcile 不会并发执行。这是设计上的保证——你不需要在 Reconcile 函数里加锁。

**不同 key 可以并行。** 通过设置 `MaxConcurrentReconciles` 控制：

```go
func (r *ClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&multiclusterv1.Cluster{}).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 5,  // 最多 5 个 goroutine 并行 reconcile 不同的 CR
        }).
        Complete(r)
}
```

默认值是 1，也就是完全串行。如果你管理几千个 CR，设成 1 会导致排队严重，延迟飙升。

**但并行不是越多越好：**

- 并行 reconcile 不同对象可能共享外部资源（如 API 限流）
- 更多 goroutine = 更多内存、更多 API Server 请求
- 通常 3-10 是合理范围，具体看你的 reconcile 函数有多重

---

## 实战场景

### 场景一：CR 卡在 Terminating 删不掉

**现象：** `kubectl delete` 后 CR 一直 Terminating，`--force --grace-period=0` 无效。

**根因：** Finalizer 存在，但负责清理的 Controller 没有运行（或清理逻辑失败）。

**排查：**

```bash
# 1. 查看 Finalizer
kubectl get <resource> <name> -o yaml | grep -A5 finalizers

# 2. 查看 Controller 状态
kubectl get pods -n <controller-namespace>

# 3. 查看 Controller 日志
kubectl logs -n <controller-namespace> <controller-pod> --tail=100
```

**解决：**

- 优先恢复 Controller 让它正常执行清理
- 确认外部资源已清理后，手动 patch 移除 Finalizer
- 长期修复：确保 Controller 有高可用部署（多副本 + leader election）

### 场景二：Mutating Webhook 挂掉，全集群阻塞

**现象：** 集群中所有 Pod 创建请求都超时失败，包括系统组件。

**根因：** 某个 Mutating Webhook 配置了 `failurePolicy: Fail`，且拦截了所有 Pod 的创建请求（比如 Istio sidecar 注入 webhook）。Webhook 的 Pod 挂了。

这里有一个经典的**死锁**：

```
Webhook Pod 挂了
→ Kubernetes 试图重新创建 Webhook Pod
→ 创建 Pod 需要经过 Mutating Webhook
→ 但 Webhook Pod 就是要被创建的那个
→ 死锁
```

**解决：**

```bash
# 紧急措施：删掉 Webhook 配置，解除阻塞
kubectl delete mutatingwebhookconfiguration <name>

# 恢复 Webhook Pod 后重新注册
```

**预防：**

1. **`failurePolicy: Ignore`** — Webhook 不可用时放行请求，而不是阻塞。但这意味着 Webhook 挂了时资源可能不符合预期。

2. **`namespaceSelector` 排除系统命名空间：**

```yaml
webhooks:
  - name: sidecar-injector.istio.io
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system", "istio-system"]
```

3. **`objectSelector` 精细匹配：** 只处理有特定 label 的资源，缩小影响面。

### 场景三：CRD 版本升级后旧 CR 读取失败

**现象：** CRD 从 v1alpha1 升级到 v1beta1 后，`kubectl get clusters` 返回部分 CR 报错。

**根因：** etcd 里存的旧 CR 是 v1alpha1 格式，新版 CRD 的 `storage` 版本是 v1beta1，但没有配置 Conversion Webhook。

**排查：**

```bash
# 查看 CRD 的 storedVersions
kubectl get crd clusters.multicluster.example.com \
  -o jsonpath='{.status.storedVersions}'
```

如果输出是 `["v1alpha1", "v1beta1"]`，说明 etcd 里同时存在两个版本的数据。

**解决：**

1. 实现 Conversion Webhook 处理 v1alpha1 ↔ v1beta1 的转换
2. 或者手动迁移所有 CR：读出 → 转换格式 → 写回
3. 迁移完成后，更新 CRD 的 `storedVersions` 移除旧版本：

```bash
# 迁移完所有 CR 后
kubectl patch crd clusters.multicluster.example.com \
  --type=json \
  -p='[{"op": "replace", "path": "/status/storedVersions", "value": ["v1beta1"]}]' \
  --subresource=status
```

**预防：** 在升级 CRD 版本前，先部署 Conversion Webhook，再修改 storage 版本，最后迁移数据。顺序很重要。

### 场景四：Controller RBAC 缺失，静默失败

**现象：** CR 创建成功，但 status 一直不更新，Controller 日志看起来也正常（没有 panic 或 crash）。

**根因：** Controller 缺少某个 RBAC 权限。比如 kubebuilder 生成的 RBAC marker 只覆盖了 CRD 本身，但 Reconcile 逻辑里还操作了 Secret、ConfigMap 等其他资源。

Controller-runtime 的默认行为是：API 调用返回 `403 Forbidden` → Reconcile 返回 error → 进入指数退避重试 → 日志里一条 `"Reconciler error"` 被淹没在几千行日志中。

**排查：**

```bash
# 查看 Controller 日志中的权限错误
kubectl logs -n <ns> <controller-pod> | grep -i "forbidden\|unauthorized\|rbac"

# 查看 Controller 的 ServiceAccount 绑定了哪些 Role
kubectl get clusterrolebinding | grep <controller-sa>
kubectl get clusterrole <role-name> -o yaml
```

**解决：**

在 Controller 代码中补上 RBAC marker，然后 `make manifests` 重新生成：

```go
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch;create;update
// +kubebuilder:rbac:groups="",resources=configmaps,verbs=get;list;watch
```

**预防：**

- 在 Reconcile 函数里对每个 API 调用做错误处理，区分权限错误和其他错误
- 添加 metrics 或 event：`recorder.Eventf(cluster, "Warning", "RBACError", "...")`
- 部署后立即测试一个完整的创建-更新-删除流程，别等上线了才发现

---

## 关键结论

- CRD 只定义数据模型（"是什么"），Controller 实现业务逻辑（"做什么"）。没有 Controller 的 CRD 就是一堆躺在 etcd 里的 JSON，不会产生任何实际效果。
- Finalizer 是一个删除保护锁：只要 `metadata.finalizers` 非空，API Server 就拒绝真正删除对象。`--force` 对 Finalizer 完全无效，因为 Finalizer 是 API Server 层面的机制，不是 kubelet 层面的。
- CR 卡在 Terminating 时，第一反应不应该是 `--force`，而是去检查 Controller 是否在运行、清理逻辑是否报错。
- Webhook 挂了可能导致死锁：Webhook Pod 挂了 → K8s 想重建它 → 创建 Pod 要经过 Webhook → 但 Webhook 就是要被创建的 → 死锁。所以 Webhook 必须用 `namespaceSelector` 排除系统命名空间。
- CRD 版本迁移的正确顺序是：先部署 Conversion Webhook，再修改 storage 版本，最后迁移数据。顺序反了会导致旧 CR 读取失败。

## 总结

CRD 是 Kubernetes 可扩展性的基石。理解 CRD + Controller + Finalizer + Webhook 的完整链路，才能在生产环境中自信地操作自定义资源。

| 概念 | 作用 | 出问题的表现 |
|---|---|---|
| CRD | 定义资源类型，API Server 自动生成 CRUD 端点 | 资源类型不存在、schema 校验失败 |
| Controller | 实现业务逻辑，驱动实际状态向期望状态收敛 | Status 不更新、外部资源不创建 |
| Finalizer | 保证删除前完成清理 | CR 卡在 Terminating |
| Mutating Webhook | 修改请求（设默认值、注入 sidecar） | 资源被意外修改、集群死锁 |
| Validating Webhook | 校验请求合法性 | 合法请求被拒绝、非法请求漏网 |
| Conversion Webhook | 多版本间数据转换 | 旧版 CR 读取失败 |

**不理解 Finalizer 的人看到 Terminating 只会 `--force`，理解 Finalizer 的人知道去查 Controller 日志。**

---

*这是「Kubernetes 深度原理实战」系列。从真实问题出发，让你理解 K8s 架构设计背后的工程智慧。*
