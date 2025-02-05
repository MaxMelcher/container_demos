# Kubernetes using aad-pod-identity to access azure key vault
https://github.com/Azure/aad-pod-identity
https://blog.jcorioland.io/archives/2018/09/05/azure-aks-active-directory-managed-identities.html

0. Variables

```
KUBE_GROUP=kubesdemo
KUBE_NAME=dzkubeaks
LOCATION="westeurope"
SUBSCRIPTION_ID=
AAD_APP_ID=
AAD_APP_SECRET=
AAD_CLIENT_ID=
TENANT_ID=
YOUR_SSH_KEY=$(cat ~/.ssh/id_rsa.pub)
SERVICE_PRINCIPAL_ID=
SERVICE_PRINCIPAL_SECRET=
ADMIN_GROUP_ID=
MY_OBJECT_ID=
KUBE_ADMIN_ID=
READER_USER_ID=

VAULT_GROUP="byokdemo"
VAULT_NAME="dzbyokdemo"
MSA_CLIENT_ID=""
MSA_PRINCIPAL_ID=""
SECRET_NAME="mySecret"
SECRET_VERSION=""
KEY_NAME="mykey"
STORAGE_NAME="dzbyokdemo"
```

1. create key vault
```
az group create -n $VAULT_GROUP -l $LOCATION
az keyvault create -n $VAULT_NAME -g $VAULT_GROUP -l $LOCATION
az keyvault create -n $VAULT_NAME -g $VAULT_GROUP -l $LOCATION --enable-soft-delete true --enable-purge-protection true --sku premium
az keyvault list -g $VAULT_GROUP -o table --query [].{Name:name,ResourceGroup:resourceGroup,PurgeProtection:properties.enablePurgeProtection,SoftDelete:properties.enableSoftDelete}
```

2. add dummy secret
```
az keyvault secret set -n $SECRET_NAME --vault-name $VAULT_NAME --value MySuperSecretThatIDontWantToShareWithYou!
```

add a key
```
az keyvault key create -n $KEY_NAME --vault-name $VAULT_NAME --kty RSA --ops encrypt decrypt wrapKey unwrapKey sign verify --protection hsm --size 2048
```
list keys
```
az keyvault key list --vault-name $VAULT_NAME -o table
```

3. create kubernetes crds
```
kubectl create -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
```

4. create managed identiy for pod
```
az identity create -n keyvaultsampleidentity -g $VAULT_GROUP

```
5. assign managed identity reader role in keyvault
```
KEYVAULT_PERMISSION="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$VAULT_GROUP/providers/Microsoft.KeyVault/vaults/$VAULT_NAME"
az role assignment create --role "Reader" --assignee $MSA_PRINCIPAL_ID --scope $KEYVAULT_PERMISSION
```

do this if the identity is not in the same resource groups as the aks nodes

```
az keyvault set-policy -n $VAULT_NAME --secret-permissions get list --spn $MSA_CLIENT_ID

KUBE_PERMISSION="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$VAULT_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/keyvaultsampleidentity"
az role assignment create --role "Managed Identity Operator" --assignee $SERVICE_PRINCIPAL_ID --scope $KUBE_PERMISSION
```

6. bind to crds

```
cat <<EOF | kubectl create -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: keyvaultsampleidentity
spec:
  type: 0
  ResourceID: /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$VAULT_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/keyvaultsampleidentity
  ClientID: $MSA_CLIENT_ID
EOF
```

7. bind pod identity

```
cat <<EOF | kubectl create -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: keyvaultsampleidentity-binding
spec:
  AzureIdentity: keyvaultsampleidentity
  Selector: keyvaultsampleidentity
EOF
```

8. deploy sample app

```
cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: demo
    aadpodidbinding: demo
  name: demo
  namespace: default
spec:
  template:
    metadata:
      labels:
        app: keyvaultsampleidentity
        aadpodidbinding: keyvaultsampleidentity
    spec:
      containers:
      - name: demo
        image: "mcr.microsoft.com/k8s/aad-pod-identity/demo:1.2"
        imagePullPolicy: Always
        args:
          - "--subscriptionid=$SUBSCRIPTION_ID"
          - "--clientid=$MSA_CLIENT_ID"
          - "--resourcegroup=$VAULT_GROUP"
          - "--aad-resourcename=https://vault.azure.net"
          # TO SPECIFY NAME OF RESOURCE TO GRANT TOKEN ADD --aad-resourcename
          # this demo defaults aad-resourcename to https://management.azure.com/
          # e.g. - "--aad-resourcename=https://vault.azure.net"
        env:
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
EOF
```

VAULT_NAME=dzdevkeyvault
AZURE_KEYVAULT_SECRET_NAME=mysupersecret
SECRET_VERSION=

key vault demo
```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: keyvaultsample
    aadpodidbinding: keyvaultsampleidentity
  name: keyvaultsample
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keyvaultsample
  template:
    metadata:
      labels:
        app: keyvaultsample
        aadpodidbinding: keyvaultsampleidentity
      name: keyvaultsample
    spec:
      containers:
      - name: keyvaultsample
        image: jcorioland/keyvault-aad-pod-identity:1.6
        env:
        - name: AZURE_KEYVAULT_NAME
          value: $VAULT_NAME
        - name: AZURE_KEYVAULT_SECRET_NAME
          value: $SECRET_NAME
        - name: AZURE_KEYVAULT_SECRET_VERSION
          value: $SECRET_VERSION
---
apiVersion: v1
kind: Service
metadata:
  name: keyvaultsample
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: keyvaultsample
  type: LoadBalancer
EOF
```

Cleanup
```
kubectl delete pods --selector=component=mic

PODNAME=`kubectl get pods --namespace=${NAMESPACE} --selector="app=tf-hub" --output=template --template="{{with index .items 0}}{{.metadata.name}}{{end}}"`
```

## create storage with byok key


1. create key vault
```
az group create -n $VAULT_GROUP -l $LOCATION
az keyvault create -n $VAULT_NAME -g $VAULT_GROUP -l $LOCATION --enable-soft-delete true --enable-purge-protection true --sku premium
az keyvault list -g $VAULT_GROUP -o table --query [].{Name:name,ResourceGroup:resourceGroup,PurgeProtection:properties.enablePurgeProtection,SoftDelete:properties.enableSoftDelete}
```

2. add a key
```
az keyvault key create -n $KEY_NAME --vault-name $VAULT_NAME --kty RSA --ops encrypt decrypt wrapKey unwrapKey sign verify --protection hsm --size 2048
```
list keys
```
az keyvault key list --vault-name $VAULT_NAME -o table
```

3. create storage account
```
az storage account create -n $STORAGE_NAME -g $VAULT_GROUP --https-only true --encryption-services table queue blob file --assign-identity
```

4. get storage account identity
```
az storage account show -g $VAULT_GROUP -n $STORAGE_NAME --query identity.principalId
```

5. set policy for storage identity to access keyvault
```
az keyvault set-policy -n $VAULT_NAME --object-id cae12840-2557-4469-909e-c29283a82c45 --key-permissions get wrapkey unwrapkey
```

6. Create or Update your Azure Storage Account to use your new keys from your Key Vault:
```
az storage account update -g $VAULT_GROUP -n $STORAGE_NAME --encryption-key-name $KEY_NAME --encryption-key-source Microsoft.KeyVault --encryption-key-vault https://$VAULT_NAME.vault.azure.net --encryption-key-version 2ce0b736baff47a4b5691edc2c53a597 --encryption-services blob file queue table
```



az acr create -n dzdemoky23 -g $VAULT_GROUP --sku Classic --location $LOCATION --storage-account-name $STORAGE_NAME

