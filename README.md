# AppName CI/CD Manifests

OpenShift 上で Backend / Frontend を Build、Test、Nexus Repository 経由の Dependency 取得、SonarQube 解析、S2I Image Build、GitOps Manifest 更新まで実行するための汎用 Tekton Manifest です。

この Repository には実在する組織名、Repository URL、クラスタ名、認証情報、顧客固有の環境設定を含めません。`example-org`、`appname-*`、`*.example.invalid` は導入先に合わせて置き換えてください。

> Current template branch: `0.1`

## Version Branch

| Branch | 内容 | 前版との差分 |
| --- | --- | --- |
| `0.1` | 基本 CI/CD 構成 | <ul><li>Backend / Frontend の Build、Test、S2I、GitOps 基盤</li></ul> |
| `1.0` | Nexus Repository / SonarQube 連携 | <ul><li>Nexus 経由の Maven / npm Dependency 取得</li><li>SonarQube Analysis と Quality Gate</li></ul> |
| `1.1` | GitOps 即時同期 | <ul><li>`update-manifest` 後に Argo CD Application Sync を実行</li><li>共用 Task を `appname-shared/` に集約</li></ul> |
| `1.2` | GitHub EventListener | <ul><li>Source Repository URL と branch による固定 Pipeline ルーティング</li><li>Webhook body から任意 Namespace / Pipeline を指定不可</li></ul> |
| `1.3` / `main` | 専用 Argo CD Instance | <ul><li>専用 Argo CD Instance と ConsoleLink を追加</li><li>小さい resource request を既定化</li></ul> |

## Directory

| Directory | 用途 |
| --- | --- |
| `appname-backend/` | Java / Maven Backend Pipeline、SonarQube、S2I Resource |
| `appname-frontend/` | Node.js / npm Frontend Pipeline、SonarQube、S2I Resource |
| `appname-shared/` | Backend / Frontend 共用 Argo CD Sync Task |
| `appname-eventlistener/` | GitHub Push EventListener と固定 TriggerTemplate |
| `appname-gitops/` | Argo CD Instance、Application、AppProject、Kustomize Sample |

## Generic Default Values

| Item | Default |
| --- | --- |
| Pipeline Namespace | `appname-pipelines-dev` |
| Application Namespace | `appname-apps-dev` |
| Argo CD Namespace | `appname-argocd-dev` |
| Backend source | `https://github.com/example-org/appname-backend.git` |
| Frontend source | `https://github.com/example-org/appname-frontend.git` |
| GitOps manifest repository | `https://github.com/example-org/appname-manifests-gitops.git` |

## Apply Order

```bash
oc apply -k appname-backend/
oc apply -k appname-frontend/
oc apply -k appname-eventlistener/
oc apply -f appname-gitops/argocd-instance.yaml
oc wait --for=jsonpath='{.status.phase}'=Available \
  argocd/appname-gitops -n appname-argocd-dev --timeout=12m
oc apply -f appname-gitops/consolelink-appname-argocd.yaml
oc apply -f appname-gitops/appproject-appname.yaml
oc apply -f appname-gitops/argocd-app-dev.yaml
oc apply -f appname-gitops/argocd-sync-rbac.yaml
```

## Security Notes

- Token、Password、Git credential は Git に保存せず、OpenShift Secret として管理してください。
- EventListener は HMAC-SHA256 を使用しません。Webhook Route はネットワーク制御または GitHub 側の送信元制限と併用してください。
- この Repository の URL、Namespace、Repository URL、Route Host はすべてサンプル値です。
