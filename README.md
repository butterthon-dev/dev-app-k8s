# このREADMEは個人的なメモです

# リソース立ち上げ方法
### 1. Artifact RegistryのリポジトリにDockerイメージをPUSH
``` shell
gcloud builds submit --tag asia-northeast1-docker.pkg.dev/cloud-learn-dev/dev-app-img-repository/sample-image:latest
```

### 2. GKEサービスアカウントバインド先のIAMサービスアカウントのcredentialをダウンロード
プロジェクトルート直下にservice_account.jsonとして配置

### 3. クラスター認証
``` shell
gcloud container clusters get-credentials cluster-dev-app --zone asia-northeast1
```

### 4. GKEサービスアカウント作成
``` shell
kubectl apply -f overlays/staging/sidecar/gke_service_account.yml
```

### 5, GKEサービスアカウントとIAMサービスアカウントの間にIAMポリシーバインディングを追加
``` shell
gcloud iam service-accounts add-iam-policy-binding \
    > gke-sa-dev-app@cloud-learn-dev.iam.gserviceaccount.com \
    > --role="roles/iam.workloadIdentityUser" \
    > --member="serviceAccount:cloud-learn-dev.svc.id.goog[default/gke-sa-dev-app]"
```

### 6. DockerイメージをPULLするためのGKEシークレット作成
``` shell
kubectl create secret docker-registry staging-artifact-registry \
    > --docker-server=https://asia-northeast1-docker.pkg.dev \
    > --docker-email=gke-sa-dev-app@cloud-learn-dev.iam.gserviceaccount.com \
    > --docker-username=_json_key \
    > --docker-password="$(cat ./overlays/staging/service_account.json)"
```

### 7. ExternalSecretをインストール
``` shell
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets
```

### 8. SecretStoreのsecretAccessKeySecretRef書き換え
k8s構成をコンソール出力
``` shell
kubectl kustomize ./overlays/staging
```

Secretのmetadata.nameを
``` shell
apiVersion: v1
data:
  secret-access-credentials: |
    ...省略...
kind: Secret
metadata:
  labels:
    stg: dev-app
  name: stg-gcpsm-secret-001-t5kgh26gcf # コレ
type: Opaque
...
..
.
```

external-secret.yml内SecretStoreのsecretAccessKeySecretRefに指定
``` dev-app-k8s/overlays/staging/patches/external-secret.yml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: secretstore
spec:
  provider:
    gcpsm:
      projectID: cloud-learn-dev
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: stg-gcpsm-secret-001-t5kgh26gcf # ココ
...
..
.
```

### 9. apply
``` shell
kubectl apply -k overlays/staging
```

# その他便利なコマンド
### Deployment再起動
``` shell
kubectl rollout restart deploy stg-dev-app-001
```
