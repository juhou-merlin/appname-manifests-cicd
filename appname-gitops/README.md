# AppName GitOps

AppName Project の開発環境を Argo CD でデプロイするための Bootstrap マニフェストです。Bootstrap の構成を AppName 用に置き換え、顧客の App Manifest Repository と CI Pipeline を接続します。

> **専用 GitOps Instance**
>
> `argocd-instance.yaml` は `appname-argocd-dev` に `appname-gitops` Argo CD Instance と専用 UI Route を作成します。AppName の Application / AppProject / Pipeline Sync RBAC はこの Namespace に配置し、共有 `openshift-gitops` Instance から分離します。Application Workload の配置先は従来どおり `appname-apps-dev` です。

## 構成

```text
appname-gitops/
├── appproject-appname.yaml       # Argo CD AppProject
├── argocd-app-dev.yaml            # 開発環境用 Application
├── argocd-instance.yaml            # AppName 専用 Argo CD Instance / Route
├── consolelink-appname-argocd.yaml # OpenShift Console Application Menu の入口
├── base/
│   ├── namespace.yaml
│   ├── backend.yaml
│   ├── frontend.yaml
│   ├── backend-config.yaml
│   └── kustomization.yaml
└── overlays/dev/
    └── kustomization.yaml         # CI が対象 Image の newTag を更新
```

## CI との接続

AppName Pipeline の `update-manifest` Task は、顧客の App Manifest Repository `EXAMPLE-ORG/appname-manifests` にある次のファイルを更新します。

この Repository 内の `base/` と `overlays/` は構成確認用の最小サンプルです。実際に Argo CD が監視する Application Manifest は、顧客 Repository の `overlays/dev` です。

```text
overlays/dev/kustomization.yaml
```

更新されるイメージは次のとおりです。

```text
appname-backend  → image-registry.openshift-image-registry.svc:5000/appname-apps-dev/appname-backend:<commit-sha>
appname-frontend → image-registry.openshift-image-registry.svc:5000/appname-apps-dev/appname-frontend:<commit-sha>
```

Pipeline が `main` で成功すると、CI が `images` 内の対象 Applicationを `name` で特定し、その要素の `newTag` だけを Commit SHA に更新して pushします。Backend実行時は `appname-backend`、Frontend実行時は `appname-frontend` だけを変更し、`newName` と他方のImage設定は変更しません。Argo CDはこの変更を検知して同期します。

## 適用順序

OpenShift GitOps Operator がインストールされたクラスタで、次の順に適用します。

```bash
oc apply -f appname-gitops/argocd-instance.yaml
oc wait --for=jsonpath='{.status.phase}'=Available \
  argocd/appname-gitops -n appname-argocd-dev --timeout=12m
oc apply -f appname-gitops/consolelink-appname-argocd.yaml
oc apply -f appname-gitops/appproject-appname.yaml
oc apply -f appname-gitops/argocd-app-dev.yaml
oc apply -f appname-gitops/argocd-sync-rbac.yaml
```

`argocd-app-dev.yaml` は顧客の `EXAMPLE-ORG/appname-manifests` Repository の `overlays/dev` を監視します。内部 Repository のため、Argo CD 用の Repository Secret を顧客標準の方法で登録してください。`appname-apps-dev` の `argocd.argoproj.io/managed-by` Label は、Argo CD CR 名ではなく Instance Namespace の `appname-argocd-dev` を値として設定します。

`consolelink-appname-argocd.yaml` は OpenShift Console の **Application Menu → Red Hat Applications → AppName Argo CD** に専用 Argo CD への入口を追加します。Route Host を変更する場合は `spec.href` も同じ Host に変更してください。

## デプロイ先

- Namespace: `appname-apps-dev`
- Argo CD Instance / Application Namespace: `appname-argocd-dev`
- Backend Service: `appname-backend:8080`
- Frontend Service: `appname-frontend:8080`
- Frontend: Node.js 24 上の Nuxt Nitro Server が SPA を配信

Backend の `/actuator/health` と Frontend の `/` を Probe に使用します。

## 注意事項

- CI の GitHub push 用 Secret `github-credentials` が必要です。
- namespace-scoped Argo CD Instance は cluster-scoped `Namespace` Resource を管理できません。Application Source には既存 Namespace 内の Resource だけを含め、Namespace は `argocd-instance.yaml` 等で事前に作成してください。
- ImageStream は CI 側の `appname-backend` と `appname-frontend` を使用します。
- `/api` の外部 Routing、認証、データベース、Production 用 overlay はこの最小構成には含めません。
- 顧客環境へ適用する前に、Route、Namespace、Registry、Proxy、GitHub Secret の値を確認してください。
