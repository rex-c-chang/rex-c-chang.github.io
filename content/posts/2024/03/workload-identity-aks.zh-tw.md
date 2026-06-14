+++
title = '在 AKS 整合 Workload Identity'
date = 2024-03-12T15:50:00+08:00
draft = false
description = "一步步整合 Azure AD Workload Identity 與 AKS，讓 Pod 透過受控識別安全存取 Azure 資源，免管理密鑰。"
tags = ["DevOps", "AKS", "Azure", "Workload Identity"]
+++

本指南將說明如何把 Azure AD Workload Identity 整合到 Azure Kubernetes Service (AKS)。透過這個整合，AKS 上的工作負載可以使用受控識別 (managed identity) 安全地存取 Azure 資源，而不需要在應用程式中直接管理密鑰或認證。

## Prerequisites

### Account Permission

開始之前，請確認以下事項：

- 你已經以使用者身分登入 Azure CLI。
- 你登入的帳號具備在 Azure 中建立使用者指派受控識別 (user-assigned managed identity) 的足夠權限。

### Tools

- Azure CLI 版本 ≥ 2.42.0
- Helm 3

### Managed Cluster

- 版本 ≥ v1.22 的 Kubernetes 叢集

## Components Installation

### OIDC

OpenID Connect 中繼資料 URL 是身分提供者 (identity provider) 的簽發者位置；Azure AD 會在權杖交換協定中使用它來驗證權杖，接著才以使用者指派受控識別的身分簽發權杖。

如果你還沒有任何叢集，執行以下指令建立：

```azurecli
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 1 \
    --enable-oidc-issuer \
    --generate-ssh-keys
```

如果已有叢集，則用以下指令更新：

```azurecli
az aks update --resource-group myResourceGroup --name myAKSCluster --enable-oidc-issuer
```

接著取得叢集的 OIDC issuer URL，稍後會用到：

```azurecli
az aks show --resource-group <resource_group> --name <cluster_name> --query "oidcIssuerProfile.issuerUrl" -otsv
```

### Mutating Admission Webhook

它具備以下功能：

- 將簽署過的 service account token 投射到一個已知的路徑。
- 依據加上註解的 service account，把與驗證相關的環境變數注入到你的 Pod。

#### Install by Azure CLI

```azurecli
az aks update --resource-group myResourceGroup --name myAKSCluster --enable-workload-identity
```

#### Install by Helm

可以參考這份[文件](https://azure.github.io/azure-workload-identity/docs/installation/mutating-admission-webhook.html)。

執行上述指令後，AKS 叢集會建立一個 workload identity controller。

## Export Environment Variables

先確認我們目前已經有的東西：

- OIDC issuer URL
- AKS 中為 Workload Identity Webhook 建立的資源

進入下一步之前，先匯出以下環境變數：

```bash
export RESOURCE_GROUP="azwi-quickstart-$(openssl rand -hex 2)"
export USER_ASSIGNED_IDENTITY_NAME="<your user-assigned managed identity name>"
export SERVICE_ACCOUNT_NAMESPACE="default"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_ISSUER="<your service account issuer URL>"
```

## Create a User-Assigned Managed Identity and Grant Permissions to Access Your Resources

建立一個將與 service account 綁定的受控識別：

```azurecli
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}"
```

接著為它指派角色。舉例來說，如果你想讓 Pod 能列出叢集憑證，可以為這個識別指派 **Azure Kubernetes Service Cluster Admin Role**。

## Create a Kubernetes Service Account

建立一個用來綁定受控識別的 service account：

```azurecli
export USER_ASSIGNED_IDENTITY_CLIENT_ID="$(az identity show --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --query 'clientId' -otsv)"
export USER_ASSIGNED_IDENTITY_OBJECT_ID="$(az identity show --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --query 'principalId' -otsv)"
```

```yml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_IDENTITY_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF
```

如果你的受控識別與 service account 位於[不同租用戶 (tenant)](https://azure.github.io/azure-workload-identity/docs/quick-start.html#5-create-a-kubernetes-service-account)，你應該替 service account 加上註解，讓 workload identity manager 能夠辨識它。

## Establish Federated Identity Credential Between the Identity and the Service Account Issuer & Subject

把 service account 與受控識別綁定：

```azurecli
az identity federated-credential create \
  --name "kubernetes-federated-credential" \
  --identity-name "${USER_ASSIGNED_IDENTITY_NAME}" \
  --resource-group "${RESOURCE_GROUP}" \
  --issuer "${SERVICE_ACCOUNT_ISSUER}" \
  --subject "system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}"
```

## Deploy Workload

```yml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quick-start
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  containers:
    - image: mcr.microsoft.com/azure-cli
      name: oidc
      command: ["tail", "-f", "/dev/null"]  # 執行 tail -f /dev/null 讓容器保持運作
      resources:
        requests:
          memory: "256Mi"   # 要求 256 MiB 記憶體
          cpu: "100m"       # 要求 100 millicpu (0.1 CPU core)
        limits:
          memory: "512Mi"   # 限制 512 MiB 記憶體
          cpu: "500m"       # 限制 500 millicpu (0.5 CPU core)
  nodeSelector:
    kubernetes.io/os: linux
EOF
```

你會看到 **AZURE_AUTHORITY_HOST**、**AZURE_CLIENT_ID**、**AZURE_TENANT_ID**、**AZURE_FEDERATED_TOKEN_FILE** 都被注入到 quick-start。各屬性的意義可參考[這份文件](https://azure.github.io/azure-workload-identity/docs/quick-start.html#7-deploy-workload)。

```bash
kubectl describe pod quick-start
```

## Verification

你可以連進 quick-start 這個 Pod，執行以下指令：

```azurecli
az login --service-principal --tenant $AZURE_TENANT_ID --federated-token "$(cat ${AZURE_FEDERATED_TOKEN_FILE})" -u $AZURE_CLIENT_ID
```

我們也可以用 init container 在一開始就取得憑證，後續再使用：

```yml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quick-start
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  initContainers:
    - name: azure-login
      image: mcr.microsoft.com/azure-cli
      command:
        - /bin/sh
        - -c
        # 1. 需要加上跳脫符號，因為環境變數不應由目前的 shell 直接展開
        # 2. 需要確保 kube config 的 ACL，否則會出現權限錯誤
        - |
          az login --service-principal --tenant \$AZURE_TENANT_ID --federated-token "\$(cat \$AZURE_FEDERATED_TOKEN_FILE)" -u \$AZURE_CLIENT_ID && \
          az aks get-credentials --resource-group myResourceGroup --name myAKSCluster --admin --file /custom/.kube/config && \
          addgroup -S myUser && \
          adduser -S myUser -G myUser && \
          chown myUser /custom/.kube/config
      volumeMounts:
        - name: aks-credentials
          mountPath: /custom
      resources:
        requests:
          memory: "256Mi"
          cpu: "200m"
        limits:
          memory: "256Mi"
          cpu: "200m"
  containers:
    - image: my-workload:latest
      name: Workload
      command: ["tail", "-f", "/dev/null"]  # 讓容器保持運作
      volumeMounts:
        - name: aks-credentials
          mountPath: /custom
      env:
        - name: KUBECONFIG
          value: /custom/.kube/config
      resources:
        requests:
          memory: "512Mi"
          cpu: "500m"
        limits:
          memory: "512Mi"
          cpu: "500m"
  volumes:
    - name: aks-credentials
      emptyDir: {}
  nodeSelector:
    kubernetes.io/os: linux
EOF
```

## References

- [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/quick-start.html)
- [Deploy and Configure Workload Identity on an Azure Kubernetes Service (AKS) Cluster](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster#create-a-managed-identity-and-grant-permissions-to-access-the-secret)
- [Arc-enabled Kubernetes and Microsoft Entra Workload ID](https://www.jannemattila.com/azure/2024/05/13/arc-enabled-kubernetes-and-entra-workload-id.html)
- [AKS Workload Identity - Quick Tutorial](https://www.youtube.com/watch?v=i2GobU0Wg48)
