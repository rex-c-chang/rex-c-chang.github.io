+++
title = 'Integrate Workload Identity with AKS'
date = 2024-03-12T15:50:00+08:00
draft = false
categories = ["DevOps"]
tags = ["AKS", "Azure", "Workload Identity"]
+++

In this guide, we will explore how to integrate Azure AD Workload Identity with Azure Kubernetes Service (AKS). This integration allows AKS workloads to securely access Azure resources using managed identities, without needing to manage secrets or credentials directly within your applications.

## Prerequisites

### Account Permission

Before we get started, ensure the following:

- You are logged in with the Azure CLI as a user.
- Your logged-in account must have sufficient permissions to create user-assigned managed identities in Azure.

### Tools

- Azure CLI version ≥ 2.42.0
- Helm 3

### Managed Cluster

- A Kubernetes cluster with version ≥ v1.22

## Components Installation

### OIDC

The OpenID Connect metadata URL of the issuer of the identity provider that Azure AD will utilize in the token exchange protocol for validating tokens before issuing a token as the user-assigned managed identity.

Run the command if you don't have any cluster:

```azurecli
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 1 \
    --enable-oidc-issuer \
    --generate-ssh-keys
```

Otherwise, update the existing cluster with the command:

```azurecli
az aks update --resource-group myResourceGroup --name myAKSCluster --enable-oidc-issuer
```

Then get your cluster's OIDC issuer URL, which will be used later:

```azurecli
az aks show --resource-group <resource_group> --name <cluster_name> --query "oidcIssuerProfile.issuerUrl" -otsv
```

### Mutating Admission Webhook

It has the following features:

- Projects a signed service account token to a well-known path.
- Injects authentication-related environment variables to your pods based on the annotated service account.

#### Install by Azure CLI

```azurecli
az aks update --resource-group myResourceGroup --name myAKSCluster --enable-workload-identity
```

#### Install by Helm

You can refer to this [document](https://azure.github.io/azure-workload-identity/docs/installation/mutating-admission-webhook.html).

The AKS cluster will create a workload identity controller once the above command is run.

## Export Environment Variables

Now let's check what we have:

- OIDC issuer URL
- Resources in AKS for Workload Identity Webhook

Before going to the next steps, we should export the following environment variables:

```bash
export RESOURCE_GROUP="azwi-quickstart-$(openssl rand -hex 2)"
export USER_ASSIGNED_IDENTITY_NAME="<your user-assigned managed identity name>"
export SERVICE_ACCOUNT_NAMESPACE="default"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_ISSUER="<your service account issuer URL>"
```

## Create a User-Assigned Managed Identity and Grant Permissions to Access Your Resources

Create a managed identity which will bind with the service account:

```azurecli
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}"
```

And assign a role for it. For example, assign **Azure Kubernetes Service Cluster Admin Role** for this identity if you want to let a pod list cluster credentials.

## Create a Kubernetes Service Account

Create a service account for binding the managed identity:

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

If your managed identity and service account are in [different tenants](https://azure.github.io/azure-workload-identity/docs/quick-start.html#5-create-a-kubernetes-service-account), you should annotate the service account to let the workload identity manager be aware of it.

## Establish Federated Identity Credential Between the Identity and the Service Account Issuer & Subject

Binding the service account and managed identity:

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
      command: ["tail", "-f", "/dev/null"]  # Run tail -f /dev/null to keep the container running
      resources:
        requests:
          memory: "256Mi"   # Request 256 MiB of memory
          cpu: "100m"       # Request 100 millicpu (0.1 CPU core)
        limits:
          memory: "512Mi"   # Limit to 512 MiB of memory
          cpu: "500m"       # Limit to 500 millicpu (0.5 CPU core)
  nodeSelector:
    kubernetes.io/os: linux
EOF
```

You will see **AZURE_AUTHORITY_HOST**, **AZURE_CLIENT_ID**, **AZURE_TENANT_ID**, and **AZURE_FEDERATED_TOKEN_FILE** have been injected into quick-start. Refer to [the doc](https://azure.github.io/azure-workload-identity/docs/quick-start.html#7-deploy-workload) to know the meaning of each attribute.

```bash
kubectl describe pod quick-start
```

## Verification

You can attach to the Pod quick-start and execute the following command:

```azurecli
az login --service-principal --tenant $AZURE_TENANT_ID --federated-token "$(cat ${AZURE_FEDERATED_TOKEN_FILE})" -u $AZURE_CLIENT_ID
```

We can also use an init container to get credentials in the beginning and use them later:

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
        # 1. It should add escape symbol because environment variables should not be interpolated by the current shell
        # 2. It should ensure ACL for kube config or it will throw permission error
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
      command: ["tail", "-f", "/dev/null"]  # Keep the container running
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
