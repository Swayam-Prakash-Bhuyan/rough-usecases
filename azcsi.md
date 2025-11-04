# Complete End-to-End Setup: Kubernetes Secret → Azure Key Vault → AKS with CSI Driver

I'll guide you through the complete setup using a **real-world Redis application** that requires secrets.

---

## 📋 **Prerequisites**

```bash
# Install Azure CLI
az --version

# Install kubectl
kubectl version --client

# Install Helm
helm version
```

---

## 🎯 **Scenario Overview**

We'll use **Bitnami Redis Helm Chart** (a real application from the internet) that requires:
- Redis password stored as a Kubernetes secret
- We'll migrate this to Azure Key Vault
- Access it via CSI driver using Service Principal authentication

---

## 📝 **Step-by-Step Implementation**

### **Step 1: Set Up Environment Variables**

```bash
# Azure Configuration
export AZURE_SUBSCRIPTION_ID="your-subscription-id"
export RESOURCE_GROUP="rg-aks-keyvault-demo"
export LOCATION="eastus"
export AKS_CLUSTER_NAME="aks-demo-cluster"
export KEY_VAULT_NAME="kv-aks-demo-$(date +%s)"  # Must be globally unique
export SERVICE_PRINCIPAL_NAME="sp-aks-keyvault-csi"

# Kubernetes Configuration
export NAMESPACE="redis-app"
export SECRET_NAME="redis-password"
```

---

### **Step 2: Create Azure Resources**

```bash
# Login to Azure
az login
az account set --subscription $AZURE_SUBSCRIPTION_ID

# Create Resource Group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Create AKS Cluster (if not exists)
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --node-count 2 \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME

# Create Azure Key Vault
az keyvault create \
  --name $KEY_VAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-rbac-authorization false  # Using access policies for SP
```

---

### **Step 3: Deploy Redis with Native Kubernetes Secret (Initial State)**

```bash
# Create namespace
kubectl create namespace $NAMESPACE

# Add Bitnami Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Generate a Redis password
export REDIS_PASSWORD=$(openssl rand -base64 32)
echo "Generated Redis Password: $REDIS_PASSWORD"

# Create Kubernetes secret (this is what we'll migrate)
kubectl create secret generic redis-password \
  --from-literal=redis-password=$REDIS_PASSWORD \
  --namespace $NAMESPACE

# Install Redis using the Kubernetes secret
helm install redis bitnami/redis \
  --namespace $NAMESPACE \
  --set auth.existingSecret=redis-password \
  --set auth.existingSecretPasswordKey=redis-password \
  --set master.persistence.enabled=false \
  --set replica.persistence.enabled=false

# Verify Redis is running
kubectl get pods -n $NAMESPACE
```

---

### **Step 4: Create Service Principal and Configure Access**

```bash
# Create Service Principal
SP_DETAILS=$(az ad sp create-for-rbac \
  --name $SERVICE_PRINCIPAL_NAME \
  --skip-assignment \
  --output json)

# Extract SP credentials
export SP_CLIENT_ID=$(echo $SP_DETAILS | jq -r '.appId')
export SP_CLIENT_SECRET=$(echo $SP_DETAILS | jq -r '.password')
export SP_OBJECT_ID=$(az ad sp show --id $SP_CLIENT_ID --query id -o tsv)
export TENANT_ID=$(echo $SP_DETAILS | jq -r '.tenant')

echo "Service Principal Created:"
echo "Client ID: $SP_CLIENT_ID"
echo "Tenant ID: $TENANT_ID"
echo "Client Secret: $SP_CLIENT_SECRET"

# Grant Key Vault access to Service Principal
az keyvault set-policy \
  --name $KEY_VAULT_NAME \
  --spn $SP_CLIENT_ID \
  --secret-permissions get list \
  --key-permissions get list \
  --certificate-permissions get list
```

---

### **Step 5: Migrate Secret from Kubernetes to Azure Key Vault**

```bash
# Extract the Redis password from Kubernetes secret
REDIS_PASSWORD=$(kubectl get secret redis-password \
  -n $NAMESPACE \
  -o jsonpath='{.data.redis-password}' | base64 -d)

# Store the password in Azure Key Vault
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name redis-password \
  --value "$REDIS_PASSWORD"

# Verify the secret in Key Vault
az keyvault secret show \
  --vault-name $KEY_VAULT_NAME \
  --name redis-password \
  --query "value" -o tsv

echo "✅ Secret migrated from Kubernetes to Azure Key Vault"
```

---

### **Step 6: Install Azure Key Vault CSI Driver**

```bash
# Install CSI driver using Helm
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm repo update

helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
  --generate-name \
  --namespace kube-system \
  --set secrets-store-csi-driver.syncSecret.enabled=true \
  --set secrets-store-csi-driver.enableSecretRotation=true

# Verify CSI driver pods are running
kubectl get pods -n kube-system -l app=csi-secrets-store-provider-azure
kubectl get pods -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver
```

---

### **Step 7: Create Service Principal Kubernetes Secret**

```bash
# Create Kubernetes secret containing SP credentials
kubectl create secret generic secrets-store-creds \
  --namespace $NAMESPACE \
  --from-literal=clientid=$SP_CLIENT_ID \
  --from-literal=clientsecret=$SP_CLIENT_SECRET

# Verify the secret
kubectl get secret secrets-store-creds -n $NAMESPACE
```

---

### **Step 8: Create SecretProviderClass**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-redis-secret
  namespace: $NAMESPACE
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    userAssignedIdentityID: ""
    keyvaultName: "$KEY_VAULT_NAME"
    cloudName: "AzurePublicCloud"
    objects: |
      array:
        - |
          objectName: redis-password
          objectType: secret
          objectVersion: ""
    tenantId: "$TENANT_ID"
  secretObjects:
  - secretName: redis-password-from-kv
    type: Opaque
    data:
    - objectName: redis-password
      key: redis-password
EOF

# Verify SecretProviderClass
kubectl get secretproviderclass -n $NAMESPACE
kubectl describe secretproviderclass azure-kv-redis-secret -n $NAMESPACE
```

---

### **Step 9: Update Redis to Use Key Vault via CSI Driver**

```bash
# Uninstall the existing Redis installation
helm uninstall redis -n $NAMESPACE

# Wait for pods to terminate
kubectl wait --for=delete pod -l app.kubernetes.io/name=redis -n $NAMESPACE --timeout=60s

# Create custom values file for Redis with CSI volume
cat <<EOF > redis-values.yaml
auth:
  existingSecret: redis-password-from-kv
  existingSecretPasswordKey: redis-password

master:
  persistence:
    enabled: false
  extraVolumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-kv-redis-secret
      nodePublishSecretRef:
        name: secrets-store-creds
  extraVolumeMounts:
  - name: secrets-store-inline
    mountPath: "/mnt/secrets-store"
    readOnly: true

replica:
  persistence:
    enabled: false
  extraVolumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-kv-redis-secret
      nodePublishSecretRef:
        name: secrets-store-creds
  extraVolumeMounts:
  - name: secrets-store-inline
    mountPath: "/mnt/secrets-store"
    readOnly: true
EOF

# Install Redis with Key Vault integration
helm install redis bitnami/redis \
  --namespace $NAMESPACE \
  --values redis-values.yaml

# Wait for Redis to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=redis -n $NAMESPACE --timeout=300s
```

---

### **Step 10: Verify the Complete Setup**

```bash
# Check if pods are running
kubectl get pods -n $NAMESPACE

# Verify CSI volume mount in Redis master pod
REDIS_MASTER_POD=$(kubectl get pod -n $NAMESPACE -l app.kubernetes.io/component=master -o jsonpath='{.items[0].metadata.name}')

echo "Checking CSI volume mount in Redis master pod:"
kubectl exec -n $NAMESPACE $REDIS_MASTER_POD -- ls -la /mnt/secrets-store/

# Verify the Kubernetes secret was synced from CSI
kubectl get secret redis-password-from-kv -n $NAMESPACE
kubectl get secret redis-password-from-kv -n $NAMESPACE -o jsonpath='{.data.redis-password}' | base64 -d
echo ""

# Test Redis connection
export REDIS_PASSWORD=$(kubectl get secret redis-password-from-kv -n $NAMESPACE -o jsonpath="{.data.redis-password}" | base64 -d)

# Port forward Redis service
kubectl port-forward -n $NAMESPACE svc/redis-master 6379:6379 &
sleep 5

# Test Redis with redis-cli (install if needed: apt-get install redis-tools)
redis-cli -h 127.0.0.1 -p 6379 -a "$REDIS_PASSWORD" ping

# Or using a test pod
kubectl run redis-test --rm -it --restart=Never \
  --namespace $NAMESPACE \
  --image=redis:latest \
  --env="REDIS_PASSWORD=$REDIS_PASSWORD" \
  --command -- redis-cli -h redis-master -a $REDIS_PASSWORD ping

# Kill port-forward
pkill -f "port-forward"
```

---

### **Step 11: Application Consumption Patterns**

#### **Pattern 1: Mount as Volume (File-based access)**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-volume-mount
  namespace: $NAMESPACE
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-kv-redis-secret
      nodePublishSecretRef:
        name: secrets-store-creds
EOF

# Verify - read secret as file
kubectl exec -n $NAMESPACE app-volume-mount -- cat /mnt/secrets/redis-password
```

#### **Pattern 2: Environment Variable (using synced K8s secret)**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-env-var
  namespace: $NAMESPACE
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sleep
    - "3600"
    env:
    - name: REDIS_PASSWORD
      valueFrom:
        secretKeyRef:
          name: redis-password-from-kv
          key: redis-password
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store-inline
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-kv-redis-secret
      nodePublishSecretRef:
        name: secrets-store-creds
EOF

# Verify - read secret as env var
kubectl exec -n $NAMESPACE app-env-var -- printenv REDIS_PASSWORD
```

---

### **Step 12: Test Secret Rotation**

```bash
# Update secret in Key Vault
NEW_PASSWORD=$(openssl rand -base64 32)
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name redis-password \
  --value "$NEW_PASSWORD"

# Wait for rotation (default: 2 minutes)
echo "Waiting for secret rotation (120 seconds)..."
sleep 120

# Verify new secret in pod
kubectl exec -n $NAMESPACE $REDIS_MASTER_POD -- cat /mnt/secrets-store/redis-password

# Check if synced K8s secret updated
kubectl get secret redis-password-from-kv -n $NAMESPACE -o jsonpath='{.data.redis-password}' | base64 -d
echo ""

echo "✅ Secret rotation verified"
```

---

### **Step 13: Monitoring and Troubleshooting**

```bash
# Check CSI driver logs
kubectl logs -n kube-system -l app=csi-secrets-store-provider-azure --tail=50

# Check secrets-store-csi-driver logs
kubectl logs -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver --tail=50

# Describe pod to see mount issues
kubectl describe pod -n $NAMESPACE $REDIS_MASTER_POD

# Check SecretProviderClass events
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp'

# Verify Key Vault access from AKS
kubectl run test-kv-access --rm -it --restart=Never \
  --namespace $NAMESPACE \
  --image=mcr.microsoft.com/azure-cli \
  --env="CLIENT_ID=$SP_CLIENT_ID" \
  --env="CLIENT_SECRET=$SP_CLIENT_SECRET" \
  --env="TENANT_ID=$TENANT_ID" \
  --env="KEY_VAULT_NAME=$KEY_VAULT_NAME" \
  --command -- bash -c "az login --service-principal -u \$CLIENT_ID -p \$CLIENT_SECRET --tenant \$TENANT_ID && az keyvault secret show --vault-name \$KEY_VAULT_NAME --name redis-password"
```

---

## 📊 **Architecture Summary**

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Key Vault                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Secret: redis-password = "actual-password"          │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Access Policy
                     │ (SP has get/list permissions)
                     │
┌────────────────────▼────────────────────────────────────────┐
│           Service Principal Authentication                   │
│  Client ID: xxx                                             │
│  Client Secret: yyy  (stored in K8s secret)                 │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│               AKS Cluster (redis-app namespace)              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ SecretProviderClass: azure-kv-redis-secret           │   │
│  │ - Provider: azure                                    │   │
│  │ - KeyVault: kv-aks-demo-xxx                         │   │
│  │ - Object: redis-password                             │   │
│  │ - Auth: Service Principal (from secrets-store-creds) │   │
│  └──────────────────────────────────────────────────────┘   │
│                     │                                         │
│                     │ CSI Driver                              │
│                     ▼                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Pod: redis-master                                    │   │
│  │ ┌────────────────────────────────────────────────┐   │   │
│  │ │ Volume Mount: /mnt/secrets-store/              │   │   │
│  │ │   └─ redis-password (file)                     │   │   │
│  │ └────────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  │ Synced K8s Secret: redis-password-from-kv          │   │
│  │   └─ Used by Redis for auth                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔒 **Security Best Practices Implemented**

1. ✅ **Service Principal with least privilege** (only get/list on secrets)
2. ✅ **SP credentials stored in Kubernetes secret** (not in pod spec)
3. ✅ **Secrets synced to K8s** for native consumption
4. ✅ **Automatic secret rotation** enabled (2-minute polling)
5. ✅ **No secrets in application code or configs**
6. ✅ **ReadOnly volume mounts**
7. ✅ **Namespace isolation**

---

## 🧹 **Cleanup**

```bash
# Delete Redis
helm uninstall redis -n $NAMESPACE

# Delete test pods
kubectl delete pod app-volume-mount app-env-var -n $NAMESPACE --ignore-not-found

# Delete namespace
kubectl delete namespace $NAMESPACE

# Uninstall CSI driver
helm list -n kube-system | grep csi-secrets-store-provider-azure | awk '{print $1}' | xargs -I {} helm uninstall {} -n kube-system

# Delete Service Principal
az ad sp delete --id $SP_CLIENT_ID

# Delete Key Vault
az keyvault delete --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP
az keyvault purge --name $KEY_VAULT_NAME

# Delete AKS (optional)
# az aks delete --name $AKS_CLUSTER_NAME --resource-group $RESOURCE_GROUP --yes --no-wait

# Delete Resource Group (if everything should be removed)
# az group delete --name $RESOURCE_GROUP --yes --no-wait
```

---

This complete setup uses **Bitnami Redis** (a real application from the Helm charts repository) and demonstrates the full lifecycle from K8s secrets to Azure Key Vault with Service Principal authentication. The Redis app requires actual secrets to function, making it a perfect real-world example! 🎉
