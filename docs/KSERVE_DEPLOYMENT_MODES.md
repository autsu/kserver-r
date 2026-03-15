# KServe 部署模式说明

本文档基于 KServe 源码梳理 **InferenceService** 与 **InferenceGraph** 支持的部署模式、配置方式及实现差异。

---

## 一、模式总览

| 模式 | 常量名 | 别名（已废弃） | 适用资源 | 工作负载由谁创建 |
|------|--------|----------------|----------|------------------|
| **Knative** | `Knative` | `Serverless` | InferenceService、InferenceGraph | KServe：创建 Knative Service |
| **Standard** | `Standard` | `RawDeployment` | InferenceService、InferenceGraph | KServe：创建 Deployment + Service + 扩缩容 |
| **ModelMesh** | `ModelMeshDeployment` | — | 仅 InferenceService | ModelMesh 控制器，KServe 只做入站/状态 |

源码中的模式定义（`pkg/constants/constants.go`）：

```go
const (
	LegacyServerless    DeploymentModeType = "Serverless"    // deprecated: use Knative
	Knative             DeploymentModeType = "Knative"
	LegacyRawDeployment DeploymentModeType = "RawDeployment" // deprecated: use Standard
	Standard            DeploymentModeType = "Standard"
	DefaultDeployment   DeploymentModeType = Standard
	ModelMeshDeployment DeploymentModeType = "ModelMesh"
)
```

支持的合法配置值（ConfigMap 校验在 `pkg/apis/serving/v1beta1/configmap.go`）：

- 合法：`Knative`、`Standard`、`ModelMesh`
- 旧值会自动归一化：`Serverless` → `Knative`，`RawDeployment` → `Standard`

---

## 二、模式如何确定

### 2.1 优先级（InferenceService / InferenceGraph）

`GetDeploymentMode`（`pkg/controller/v1beta1/inferenceservice/utils/utils.go`）逻辑：

1. **最高**：`status.deploymentMode`（已写入 status 的解析结果）
2. **其次**：注解 `serving.kserve.io/deploymentMode`（若为合法值）
3. **默认**：ConfigMap `inferenceservice-config` 的 `deploy.defaultDeploymentMode`

注解 key 为常量 `constants.DeploymentMode`（`serving.kserve.io/deploymentMode`）。

### 2.2 默认值配置

- ConfigMap：`inferenceservice-config`，data key：`deploy`。
- 结构（`pkg/apis/serving/v1beta1/configmap.go`）：

```go
type DeployConfig struct {
	DefaultDeploymentMode     string                     `json:"defaultDeploymentMode,omitempty"`
	DeploymentRolloutStrategy *DeploymentRolloutStrategy `json:"deploymentRolloutStrategy,omitempty"`
}
```

- `defaultDeploymentMode` 必填，且只能是 `Knative`、`Standard`、`ModelMesh` 之一。

---

## 三、三种模式的区别（结合源码）

### 3.1 Knative 模式

**依赖**：集群需安装 Knative Serving（CRD `Service.serving.knative.dev/v1`）。未安装时 controller 会报错并终止调和（`ServerlessModeRejected`）。

**工作负载**（Predictor/Transformer/Explainer）：

- 不创建 Kubernetes `Deployment`，改为创建 **Knative Service**（`knative.dev/serving`）。
- 入口：`Predictor.reconcileKnativeDeployment` → `knative.NewKsvcReconciler(...).Reconcile(ctx)`（`pkg/controller/v1beta1/inferenceservice/components/predictor.go`）。

**入站与扩缩容**：

- **Ingress**：与 ModelMesh 共用同一套逻辑，使用「Knative 入站」Reconciler（`ingress.NewIngressReconciler`），创建 **Istio VirtualService** 等，将流量指到 Knative Service。
- **扩缩容**：由 Knative 自身（KPA/autoscaler）负责，KServe 不创建 HPA/KEDA。

**特点**：

- 支持缩容到 0（取决于 Knative 配置）。
- 不涉及 `Deployment`、`deploymentStrategy`、KServe 侧的 HPA/KEDA。

---

### 3.2 Standard 模式（Raw / 原生 K8s）

**依赖**：无需 Knative 或 ModelMesh，仅需标准 Kubernetes。

**工作负载**：

- 为每个组件（Predictor、Transformer、Explainer）创建：
  - **Deployment**（及可选的 worker Deployment）
  - **Service**
  - 扩缩容：**HPA** 或 **KEDA ScaledObject**（若集群有对应 CRD）
- 入口：`Predictor.reconcileRawDeployment` → `RawKubeReconciler`（`pkg/controller/v1beta1/inferenceservice/reconcilers/raw/raw_kube_reconciler.go`）。

**Reconciler 工厂**（`pkg/controller/v1beta1/inferenceservice/reconcilers/factory.go`）：

- **Workload**：`deployment.NewDeploymentReconciler` → 只处理 `Standard` / `LegacyRawDeployment`。
- **Service**：`service.NewServiceReconciler` → 同上。
- **Ingress**：
  - 若启用 Gateway API：`ingress.NewRawHTTPRouteReconciler`（HTTPRoute）。
  - 否则：`ingress.NewRawIngressReconciler`（K8s Ingress）。

**仅在此模式生效的配置**：

- **deploymentStrategy**：`ComponentExtensionSpec.DeploymentStrategy`（如 RollingUpdate / Recreate）。校验在 `inference_service_validation.go` 中限定为 raw 模式。
- **deploymentRolloutStrategy**：ConfigMap 中 `DeployConfig.DeploymentRolloutStrategy`（默认 rollout 行为）。

**特点**：

- 纯 K8s 资源，易排障、易与现有 K8s 工具链集成。
- 可显式配置 HPA/KEDA、Deployment 策略。

---

### 3.3 ModelMesh 模式

**依赖**：集群中由 **ModelMesh** 管理模型运行时；KServe 不创建 Predictor 的 Deployment/Knative Service。

**行为**（`pkg/controller/v1beta1/inferenceservice/controller.go`）：

- **仅 Predictor、无 Transformer**：  
  - 直接跳过 KServe 的调和（不创建 workload），只做 status 初始化。
- **有 Transformer**：  
  - 继续调和 Transformer（Transformer 仍由 KServe 以 Knative 或 Raw 方式创建，取决于该 ISVC 上实际解析出的 mode；入站与 Knative/Standard 一致地走 IngressReconciler 或 Raw Ingress/HTTPRoute）。

**入站**：

- 与 Knative 相同：使用 `ingress.NewIngressReconciler`（VirtualService 等），指向 Knative Service 或 ModelMesh 暴露的端点。

**状态**：

- 若为 ModelMesh，status 比较时会缩小范围，排除由 ModelMesh 管理的 Predictor/Model 相关字段（见 controller 中 `deploymentMode == constants.ModelMeshDeployment` 时的 status 比较逻辑）。

**特点**：

- Predictor 由 ModelMesh 管理，适合多模型、高密度部署。
- KServe 只负责入站与（有 Transformer 时的）Transformer 生命周期。

---

## 四、InferenceGraph 与 InferenceService 的差异

- **InferenceGraph** 只支持两种模式：
  - **Standard**：创建 Deployment + Service + HPA 等（`handleInferenceGraphRawDeployment`）。
  - **Knative**：创建 Knative Service（`createKnativeService` + `GraphKnativeServiceReconciler`）。
- **InferenceGraph 没有 ModelMesh 模式**；ModelMesh 仅用于 InferenceService。

---

## 五、源码路径速查

| 内容 | 路径 |
|------|------|
| 模式常量与解析 | `pkg/constants/constants.go`（`DeploymentModeType`、`ParseDeploymentMode`） |
| 模式解析（status/annotation/config 优先级） | `pkg/controller/v1beta1/inferenceservice/utils/utils.go`（`GetDeploymentMode`） |
| DeployConfig / 合法模式校验 | `pkg/apis/serving/v1beta1/configmap.go`（`NewDeployConfig`） |
| 默认注解写入 | `pkg/apis/serving/v1beta1/inference_service_defaults.go`（`DefaultInferenceService`） |
| Workload/Service/Ingress 按模式分支 | `pkg/controller/v1beta1/inferenceservice/reconcilers/factory.go` |
| Standard 模式：Deployment+Service+Scaler | `pkg/controller/v1beta1/inferenceservice/reconcilers/raw/raw_kube_reconciler.go` |
| Knative 模式：Knative Service | `pkg/controller/v1beta1/inferenceservice/components/predictor.go`（`reconcileKnativeDeployment`）及 knative reconciler |
| ModelMesh 跳过调和与 status | `pkg/controller/v1beta1/inferenceservice/controller.go`（ModelMesh 分支） |
| deploymentStrategy 仅 raw | `pkg/apis/serving/v1beta1/inference_service_validation.go`、`component.go` |
| InferenceGraph Standard/Knative | `pkg/controller/v1alpha1/inferencegraph/controller.go` |

---

## 六、小结

| 维度 | Knative | Standard | ModelMesh |
|------|---------|----------|-----------|
| 工作负载 | Knative Service | Deployment + Service + HPA/KEDA | ModelMesh 管理 Predictor |
| 入站 | VirtualService（IngressReconciler） | Ingress 或 HTTPRoute（Raw） | 同 Knative（VirtualService） |
| 扩缩容 | Knative KPA | HPA / KEDA | ModelMesh / 其他 |
| deploymentStrategy | 不支持 | 支持 | 不适用 |
| 依赖 | Knative Serving | 无 | ModelMesh |
| 适用资源 | ISVC、IG | ISVC、IG | 仅 ISVC |

通过注解 `serving.kserve.io/deploymentMode` 或 ConfigMap `defaultDeploymentMode` 选择模式；旧值 `Serverless`/`RawDeployment` 会在解析时自动映射为 `Knative`/`Standard`。
