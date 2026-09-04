# AppName Frontend CI

## 用途

このディレクトリは AppName Frontend（Nuxt 4、Vue 3、TypeScript、Node.js 24、`ssr:false`）の CI 専用マニフェストです。既存の Sample ディレクトリには依存しません。

Kustomize 適用先 namespace は `appname-pipelines-dev` です。ImageStream、BuildConfig、Pipeline、Task、PVC、ServiceAccount はこの namespace に作成されます。

## 流れ

`git-clone` → `npm ci` → `npm run build` → `npm run test:coverage` → `npm run typecheck` → Runtime Artifact 準備 →（main のみ）OpenShift Node.js Binary Source-to-Image Build → Internal Registry

Build/Test では `npm run test`、`format:check`、SonarQube、Playwright E2E、Backend/PostgreSQL/SMTP/RustFS の起動は行いません。

## Runtime Artifact と本物の S2I

`npm-build-test.yaml` は `.output/server/index.mjs` と `.output/public` を検証し、`.output` と起動用の最小 `package.json` を `.s2i-input` に準備します。`s2i-build-task.yaml` は Frontend Source 全体ではなく、この Runtime Artifact だけを `oc start-build --from-dir` で `appname-frontend-s2i` に渡します。

BuildConfig は `sourceStrategy` と `registry.access.redhat.com/ubi9/nodejs-24:latest` を使用します。Runtime Artifact の `package.json` には `build` Script を含めないため、Node.js S2I は Nuxt Build を繰り返さず、Image 起動時に `npm start` から `node .output/server/index.mjs` を実行します。Tekton は Buildah や Dockerfile を使用しません。

Application Source の `npm run build` は `nuxt build` を実行し、`.output/server/index.mjs` を生成する必要があります。`nuxt generate` により静的ファイルだけを生成する構成では、この Task は明確に失敗します。

## SPA / `/api` の注意

Production の Nitro Server には Nuxt の開発用 `devProxy` はありません。Backend Service 名、環境固有 URL、`localhost:8082` はこの CI Manifest に埋め込みません。Production の `/api` Routing（OpenShift Route の Path Routing、Gateway/Ingress 等）は後続の CD/Application Manifest で決定します。

## 外部準備

- `pipeline.yaml`: デフォルト Repository は `https://github.com/example-org/appname-frontend.git`。必要に応じて PipelineRun から上書きする。
- `s2i-build-task.yaml`: `oc` CLI は OCP 4.22 向けの `quay.io/openshift/origin-cli:4.22` を使用する。PipelineRun は namespace のデフォルト `pipeline` ServiceAccount を使用する。
- Application Source: `npm run build` が `nuxt build` を実行し、`.output/server/index.mjs` を生成することを確認する。

## PVC / RBAC

`appname-frontend-pipeline-pvc`（RWO、5Gi）を source と npm cache に使用します。namespace のデフォルト `pipeline` ServiceAccount に、Binary Build 起動、Build 状態/ログ参照、ImageStream Tag 操作だけを許可します。privileged SCC、root、cluster-admin は付与していません。

## 実行前チェックと検証

1. Repository URL、対象 namespace、`oc` イメージを確認する。
2. `git-clone` Cluster Task が `openshift-pipelines` に存在することを確認する。
3. Nuxt の Build 出力に `.output/server/index.mjs` と `.output/public` が含まれることを実 Repository で確認する。
4. main で実行し、BuildConfig の `latest`、Commit SHA Tag、Pipeline Result `IMAGE_URL` / `IMAGE_TAG` を確認する。非 main では Image Build と Image 関連の Pipeline Result は使用しません。

RWO PVC を使用する場合は TektonConfig の `spec.pipeline.coschedule=disabled` を設定します。PipelineRun は同じ PVC を source と npm cache の二つの workspace に割り当て、同一アプリの PipelineRun を同時実行しない前提です。

## GitOps との接続

main で Image Build が成功すると、Backend と共用する `git-update-manifest` Task が `EXAMPLE-ORG/appname-manifests` Repository の `overlays/dev/kustomization.yaml` にある `appname-frontend` の `newTag` だけを Commit SHA に更新して push します。Argo CD の Application 定義は `appname-gitops/` にあります。

EventListener、Webhook、Pipelines as Code、SonarQube、Playwright E2E、環境間 Promotion、Production `/api` Routing は後続 Task です。
