# AppName Backend CI

## 用途

このディレクトリは AppName Backend（Spring Boot 3.5.16、Java 25）の CI 専用マニフェストです。既存の Sample ディレクトリには依存しません。

Kustomize 適用先 namespace は `appname-pipelines-dev` です。ImageStream、BuildConfig、Pipeline、Task、PVC、ServiceAccount はこの namespace に作成されます。

## 流れ

`git-clone` → `./mvnw -B clean verify` →（main のみ）OpenShift Binary Source-to-Image Build → Internal Registry

`verify` が Unit Test、Checkstyle、SpotBugs、JaCoCo、Package を実行します。SonarQube は実行しません。

## Sample との差分

Sample と同じく cluster resolver の `git-clone`、`source` サブディレクトリ、PVC 上の Maven cache、Commit SHA の利用、main 分岐の `when` を使います。Maven は OpenJDK 25 に変更し、Buildah、動的 Dockerfile、Manifest 更新、共有 privileged RBAC は使いません。

## 本物の S2I

`s2i-buildconfig.yaml` の `appname-backend-s2i` が `sourceStrategy` と `registry.access.redhat.com/ubi9/openjdk-25:latest` を定義します。`s2i-build-task.yaml` は `oc start-build --from-dir` で同じ Clone 済みソースを Binary Input として渡し、Build 完了後に `latest` から Commit SHA Tag を追加します。Tekton 自身は Buildah や `s2i` CLI を実行しません。

Maven settings は Task と OpenShift Build の両方で同じ ConfigMap を参照します。OpenShift Build では `spec.source.configMaps` により `settings.xml` を Build 時だけ `configuration` へコピーします。読み取り専用 Volume を Maven ディレクトリへ Mount しないため、OpenJDK 25 S2I は設定ファイルを処理できます。また、`MAVEN_ARGS` は上書きせず、Builder Image の既定値を維持します。Maven Repository 用の Proxy は `maven-proxy-settings.yaml` の `<proxy>` に設定します。Cluster-wide Proxy が設定された環境では、Tekton Task Pod の Git/Maven/npm/oc 通信と OpenShift Build は自動注入された Proxy を使用します。

## 外部準備

- `pipeline.yaml`: デフォルト Repository は `https://github.com/example-org/appname-backend.git`。必要に応じて PipelineRun から上書きする。
- `git-update-manifest.yaml`: Frontend と共用する AppName GitOps 更新 Task。更新先は顧客 App Manifest Repository の `overlays/dev`。
- `s2i-build-task.yaml`: `oc` CLI は OCP 4.22 向けの `quay.io/openshift/origin-cli:4.22` を使用する。PipelineRun は namespace のデフォルト `pipeline` ServiceAccount を使用する。
- `maven-proxy-settings.yaml`: Maven Repository 用の顧客 Proxy のホスト/ポートを `<proxy>` に設定する。

## PVC / RBAC

`appname-backend-pipeline-pvc`（RWO、5Gi）を source と Maven cache に使用します。namespace のデフォルト `pipeline` ServiceAccount に、BuildConfig Binary 起動、Build 状態/ログ参照、ImageStream Tag 操作だけを許可します。privileged SCC、root、cluster-admin は付与していません。

## 実行前チェック

1. Repository URL、Proxy の実ホスト/ポート、ConfigMap/Secret 方針、`oc` イメージ、対象 namespace（`appname-pipelines-dev`）を確認する。
2. `git-clone` Cluster Task が `openshift-pipelines` に存在することを確認する。
3. 対象 OCP で ConfigMap Build Source と OpenJDK 25 S2I Builder の `configuration/settings.xml` 読み込みを確認する。
4. namespace の `pipeline` ServiceAccount と 5Gi RWO PVC が利用できることを確認する。

## 検証

main で実行した場合、BuildConfig の `latest` と同じ ImageStream の Commit SHA Tag、および Pipeline Result `IMAGE_URL` / `IMAGE_TAG` を確認します。非 main では Image Build と Image 関連の Pipeline Result は使用しません。

RWO PVC を使用する場合は TektonConfig の `spec.pipeline.coschedule=disabled` を設定します。PipelineRun は同じ PVC を source と Maven cache の二つの workspace に割り当て、同一アプリの PipelineRun を同時実行しない前提です。

## GitOps との接続

main で Image Build が成功すると、共用の `git-update-manifest` Task が `EXAMPLE-ORG/appname-manifests` Repository の `overlays/dev/kustomization.yaml` にある `appname-backend` の `newTag` だけを Commit SHA に更新して push します。Argo CD の Application 定義は `appname-gitops/` にあります。

EventListener、Webhook、Pipelines as Code、SonarQube、E2E、環境間 Promotion は後続 Task です。
