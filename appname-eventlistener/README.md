# AppName GitHub EventListener（测试环境）

本目录部署一个统一的 GitHub Webhook 入口。它不进行 GitHub HMAC-SHA256 签名验证，仅允许下表中列出的 `main` 分支仓库触发对应的固定 Pipeline。

| Source repository | Pipeline namespace | Pipeline |
| --- | --- | --- |
| `https://github.com/example-org/appname-backend.git` | `appname-pipelines-dev` | `appname-backend-pipeline` |
| `https://github.com/example-org/appname-frontend.git` | `appname-pipelines-dev` | `appname-frontend-pipeline` |

请求体中的 Repository URL 只用于选择已定义的 Trigger；它**不能**指定任意的 Pipeline、Namespace、GitOps Repository 或 Argo CD Application。

## 部署

部署并取得 HTTPS URL：

```bash
oc apply -k appname-eventlistener/
oc get route cicd-github-webhook -n appname-pipelines-dev
```

在两个测试 Source Repository 的 GitHub Webhook 中分别登记该 HTTPS URL，设置：

- Content type: `application/json`
- Secret: 不设置
- Event: `Just the push event`

## 未来系统扩展

定例资料中的逻辑环境名称为 `AppName`、`BUSINESS-UNIT`、`NEW-LOGISTICS`。Kubernetes Namespace 不能使用日文，因此实际 Namespace 采用以下 ASCII 名称：

| 资料中的系统 | Pipeline Namespace | GitOps/Argo CD Namespace |
| --- | --- | --- |
| AppName | `appname-pipelines-dev` | `appname-argocd-dev` |
| BUSINESS-UNIT | `bit-pipelines-dev` | `bit-argocd-dev` |
| NEW-LOGISTICS | `new-logistics-pipelines-dev` | `new-logistics-argocd-dev` |

追加系统时复制一个 TriggerTemplate 和 Trigger，并把该系统的 Source Repository URL、目标 Pipeline 名、目标 Namespace、GitOps Repository 和 Argo CD Application 写死在模板中。同时仅在该目标 Namespace 向 `cicd-eventlistener` 授予创建该 PipelineRun 所需的最小权限。不要从 Webhook body 读取这些目标值。HMAC を使用しないため、Webhook URL は社内ネットワークや GitHub 側の送信元制限など、環境のアクセス制御と併用してください。
