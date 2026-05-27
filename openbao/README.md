# OpenBao
(From [OpenBao Official Website](https://openbao.org/))

*OpenBao is an open source, community-driven secrets manager and fork of Hashicorp Vault managed by the Linux Foundation's OpenSSF.*

*Use cases:*
1. *Secure Secret Storage*
2. *Dynamic Secrets*
3. *Data Encryption*
4. *Identity based access*
5. *Leasing and Renewal*
6. *Revocation*

OpenBao is used in this sandbox as a way for pods to fetch Vaulted secrets as Environment Variables or VolumeMounts by using [Secrets CSI](https://github.com/kubernetes-sigs/secrets-store-csi-driver). 

The installation of OpenBao is done using minimal values and Dev Mode: an ephemeral pod that loses all config once it is killed/restarted. 

## Install secrets CSI
```sh
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm repo update

helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  -n kube-system \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true \
  --set rotationPollInterval=30s
```

## Install OpenBao (with Secrets CSI enabled)
```sh
helm repo add openbao https://openbao.github.io/openbao-helm

helm repo update

helm install openbao openbao/openbao \
--set "server.dev.enabled=true" \
--set "csi.enabled=true"
```

### Configure a SA Token example

1. Create a minimal Cluster Role for OpenBao to manage Kubernetes SA Tokens 
```sh
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-minimal-secrets-abilities
rules:
- apiGroups: [""]
  resources: ["serviceaccounts/token"]
  verbs: ["create"]
EOF

kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openbao-token-creator-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k8s-minimal-secrets-abilities
subjects:
- kind: ServiceAccount
  name: openbao
  namespace: default
EOF
```

2. Create Namespace and Service Account and the appropriate permissions

```sh
kubectl create namespace openbao-sa-example
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-account-with-generated-token
  namespace: openbao-sa-example
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-list-pods
  namespace: openbao-sa-example
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-abilities
  namespace: openbao-sa-example
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: role-list-pods
subjects:
- kind: ServiceAccount
  name: service-account-with-generated-token
  namespace: openbao-sa-example
EOF
```

3. Enable kubernetes secrets

```sh
bao secrets enable kubernetes
bao write -f kubernetes/config
```

4. Create the Kubernetes Role
```sh
bao write kubernetes/roles/my-role \
    allowed_kubernetes_namespaces="openbao-sa-example" \
    service_account_name="service-account-with-generated-token" \
    token_default_ttl="10m"
```

5. Create a Token
```sh
bao write kubernetes/creds/my-role \
    kubernetes_namespace=openbao-sa-example
# This will generate your TOKEN to use on the next step and is only available for view once
```

6. Test an API call

```sh
NAMESPACE=openbao-sa-example
TOKEN=yourbearertoken

kubectl run nginx --image=nginx -n $NAMESPACE

curl -sk $(kubectl config view --minify -o 'jsonpath={.clusters[].cluster.server}')/api/v1/namespaces/$NAMESPACE/pods \
    --header "Authorization: Bearer $TOKEN"
```

## Pulling secrets from OpenBao

1. Create OpenBao Policy
```sh
bao policy write bao-secrets-read - <<EOF
path "secret/data/database/creds/db-app" {
  capabilities = ["read"]
}
EOF
```

2. Enable Kubernetes Auth
```sh
bao auth enable kubernetes
bao write auth/kubernetes/config \
    kubernetes_host='https://kubernetes.default.svc.cluster.local'
```

3. Create the OpenBao Auth Role
```sh
bao write auth/kubernetes/role/bao-secrets-role \
    bound_service_account_names=bao-secrets-sa \
    bound_service_account_namespaces=bao-secrets \
    policies=bao-secrets-read \
    ttl=1h
```

4. Configure the Namespace and Service Account to fetch the secrets
```sh
kubectl create namespace bao-secrets
```

```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bao-secrets-sa
  namespace: bao-secrets
EOF
```

5. Create the OpenBao Secret
```sh
# Execute this command with OpenBao directly on pod or your CLI (requires login and endpoint configured)
bao kv put secret/database/creds/db-app \
  username=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16; echo) \
  password=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 32; echo)
```

### Configure a pod to pull a secret from OpenBao as Volumes

1. Create the SecretProviderClass values to be accessible as volumes
```sh
kubectl apply -f - <<EOF
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: openbao-db-creds-vol
  namespace: bao-secrets
spec:
  provider: openbao
  parameters:
    vaultAddress: "http://openbao.default:8200"
    roleName: bao-secrets-role
    objects: |
      - objectName: "username"
        secretPath: "secret/data/database/creds/db-app"
        secretKey: "username"
      - objectName: "password"
        secretPath: "secret/data/database/creds/db-app"
        secretKey: "password"
EOF
```

2. Create the Deployment using volume mounts
```sh
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # Reloader annotation for CSI-mounted secrets (file-based)
    secretproviderclass.reloader.stakater.com/reload: "openbao-db-creds-vol"
  labels:
    app: bao-vol-deploy
  name: bao-vol-deploy
  namespace: bao-secrets
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bao-vol-deploy
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: bao-vol-deploy
    spec:
      serviceAccountName: bao-secrets-sa
      containers:
        - image: nginx
          name: nginx
          volumeMounts:
            - name: 'openbao-db-creds'
              mountPath: '/mnt/secrets-store'
              readOnly: true
      volumes:
        - name: openbao-db-creds
          csi:
            driver: 'secrets-store.csi.k8s.io'
            readOnly: true
            volumeAttributes:
              secretProviderClass: 'openbao-db-creds-vol'
EOF
```

### Configure a pod to pull a secret from OpenBao as Environment Variable

1. Create the SecretProviderClass values to be accessible as environment variables
```sh
kubectl apply -f - <<EOF
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: openbao-db-creds-env
  namespace: bao-secrets
spec:
  provider: openbao
  secretObjects:
    - secretName: openbao-db-creds-secret-env
      type: Opaque
      data:
        - objectName: username
          key: username
        - objectName: password
          key: password
  parameters:
    vaultAddress: "http://openbao.default.svc.cluster.local:8200"
    roleName: bao-secrets-role
    objects: |
      - objectName: "username"
        secretPath: "secret/data/database/creds/db-app"
        secretKey: "username"
      - objectName: "password"
        secretPath: "secret/data/database/creds/db-app"
        secretKey: "password"
EOF
```

2. Create the Deployment using environment variables
```sh
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bao-env-deploy
  namespace: bao-secrets
  labels:
    app: bao-env-deploy
spec:
  selector:
    matchLabels:
      app: bao-env-deploy
  replicas: 1
  template:
    metadata:
      labels:
        app: bao-env-deploy
    spec:
      serviceAccountName: bao-secrets-sa
      containers:
      - image: nginx
        name: nginx
        env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: openbao-db-creds-secret-env
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: openbao-db-creds-secret-env
                  key: password
        volumeMounts:
          - name: 'openbao-db-creds'
            mountPath: '/mnt/secrets-store'
            readOnly: true
      volumes:
        - name: openbao-db-creds
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: 'openbao-db-creds-env'
EOF
```

## Useful Links
- [Helm chart | OpenBao](https://openbao.org/docs/platform/k8s/helm/)
- [openbao-helm/charts/openbao/values.yaml at main · openbao/openbao-helm](https://github.com/openbao/openbao-helm/blob/main/charts/openbao/values.yaml)
- [Kubernetes secrets engine | OpenBao](https://openbao.org/docs/secrets/kubernetes/)
- [Kubernetes auth method | OpenBao](https://openbao.org/docs/auth/kubernetes/)
- [CSI Driver - Reloader](https://docs.stakater.com/reloader/latest/integrations/openbao/openbao-csi.html)
- [CSI Driver (File-Based) - Reloader](https://docs.stakater.com/reloader/latest/integrations/openbao/openbao-csi-file.html)