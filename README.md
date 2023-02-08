# cosign-tutorial

### 環境変数
```
export PROJECT_ID=linkerd-sandbox
export REGION=asia-northeast1
export ZONE=asia-northeast1-c
export KEY_RING=cosign
export KEY_NAME=cosign
export REGISTRY_NAME=containers
export CLUSTER_NAME=cosign-test-cluster
export GSA_NAME=${CLUSTER_NAME}-sa
export GSA_ID=${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
roles="roles/logging.logWriter roles/monitoring.metricWriter roles/monitoring.viewer roles/cloudkms.viewer roles/cloudkms.verifier"
```

### プロジェクト認証
```
gcloud config set project ${PROJECT_ID}
gcloud services enable cloudkms.googleapis.com
```

```
gcloud kms keyrings create ${KEY_RING} --location ${REGION}
gcloud kms keys create ${KEY_NAME} \
    --keyring ${KEY_RING} \
    --location ${REGION} \
    --purpose asymmetric-signing \
    --default-algorithm ec-sign-p256-sha256
gcloud services enable artifactregistry.googleapis.com
gcloud artifacts repositories create ${REGISTRY_NAME} \
    --repository-format docker \
    --location ${REGION}
```

### Image を GAR に Push する
```
docker pull nginx
docker tag nginx ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx
gcloud auth configure-docker ${REGION}-docker.pkg.dev
SHA=$(docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx | grep digest: | cut -f3 -d" ")
```

### Cosign をインストールする
```
apt update
apt-get install -y jq
COSIGN_VERSION=$(curl -s https://api.github.com/repos/sigstore/cosign/releases/latest | jq -r .tag_name)
curl -LO https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64
mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
```

```
gcloud auth application-default login
export SHA="sha256:7fd1b0f92598e73e13d996d6ce279905ac5f2381d635c712a86bf8e49cabf0be"
cosign generate-key-pair \
    --kms gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEY_RING}/cryptoKeys/${KEY_NAME}

gcloud auth configure-docker asia-northeast1-docker.pkg.dev

cosign sign \
    --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEY_RING}/cryptoKeys/${KEY_NAME} \
    ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx@${SHA}

gcloud artifacts docker tags list ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx
cosign verify \
    --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEY_RING}/cryptoKeys/${KEY_NAME} \
    ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx@${SHA}
```

```
root@ec9e9d9e325c:/# cosign verify \
    --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEY_RING}/cryptoKeys/${KEY_NAME} \
    ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx@${SHA}

Verification for asia-northeast1-docker.pkg.dev/linkerd-sandbox/containers/nginx@sha256:7fd1b0f92598e73e13d996d6ce279905ac5f2381d635c712a86bf8e49cabf0be --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"asia-northeast1-docker.pkg.dev/linkerd-sandbox/containers/nginx"},"image":{"docker-manifest-digest":"sha256:7fd1b0f92598e73e13d996d6ce279905ac5f2381d635c712a86bf8e49cabf0be"},"type":"cosign container image signature"},"optional":null}]
root@ec9e9d9e325c:/# 
```

# Cluster を作成する
```
gcloud iam service-accounts create ${GSA_NAME} \
    --display-name ${GSA_NAME}

for role in $roles; do gcloud projects add-iam-policy-binding ${PROJECT_ID} --member "serviceAccount:${GSA_ID}" --role $role; done
gcloud artifacts repositories add-iam-policy-binding ${REGISTRY_NAME} \
    --location ${REGION} \
    --member "serviceAccount:${GSA_ID}" \
    --role roles/artifactregistry.reader
gcloud services enable container.googleapis.com
gcloud beta container --project "linkerd-sandbox" clusters create "cosign-b" --zone "asia-northeast1-c" --no-enable-basic-auth --cluster-version "1.24.8-gke.2000" --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/cloudkms" --max-pods-per-node "110" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/linkerd-sandbox/global/networks/default" --subnetwork "projects/linkerd-sandbox/regions/asia-northeast1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "asia-northeast1-c"

gcloud beta container --project "linkerd-sandbox" node-pools create "pool-1" --cluster "cosign-b" --zone "asia-northeast1-c" --node-version "1.24.8-gke.2000" --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/cloudkms" --num-nodes "1" --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --max-pods-per-node "110"
```

### Policy Controller をインストールする
```
helm repo add sigstore https://sigstore.github.io/helm-charts
helm repo update
helm install policy-controller \
    -n cosign-system sigstore/policy-controller \
    --create-namespace
```

 ### 署名の取得元を定義する
```
cat << EOF | kubectl apply -f -
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: private-signed-images-cip
spec:
  images:
  - glob: "**"
  authorities:
  - key:
      kms: gcpkms://projects/linkerd-sandbox/locations/asia-northeast1/keyRings/cosign/cryptoKeys/cosign/cryptoKeyVersions/1
EOF
```

### Policy を有効化する
```
kubectl create namespace test
kubectl label namespace test policy.sigstore.dev/include=true
```

### tag を指定しない
```
yamamoto_daisuke@cloudshell:~ (linkerd-sandbox)$ kubectl create deployment nginx \
    --image=nginx \
    -n test
error: failed to create deployment: admission webhook "policy.sigstore.dev" denied the request: validation failed: failed policy: private-signed-images-cip: spec.template.spec.containers[0].image
index.docker.io/library/nginx@sha256:c54fb26749e49dc2df77c6155e8b5f0f78b781b7f0eadd96ecfabdcdfa5b1ec4 signature key validation failed for authority authority-0 for index.docker.io/library/nginx@sha256:c54fb26749e49dc2df77c6155e8b5f0f78b781b7f0eadd96ecfabdcdfa5b1ec4: no matching signatures:
```

### tag を指定する
```
yamamoto_daisuke@cloudshell:~ (linkerd-sandbox)$ kubectl create deployment nginx \
    --image=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/nginx@${SHA} \
    -n test
deployment.apps/nginx created
yamamoto_daisuke@cloudshell:~ (linkerd-sandbox)$
```
