# arc-gitops-argocd
Use ArgoCD with Arc-enabled Kubernetes clusters

Azure Arc for Kubernetes leverages Flux (v2) to manage clusters in a GitOps fashion. But what if you would like to use ArgoCD? This repo will help you out. Just dropped any manifest (ArgoCD Application CRD or other Kubernetes manifets) in the `argocd` folder and they will be installed in your cluster; it uses Flux first to deploy ArgoCD and a Root App that points to this very repository.

First create an AKS cluster. The way I do it is this bu feel free to use any other combination of options:

```
az aks create  \
--load-balancer-sku standard --enable-managed-identity \
-l westeurope -k 1.22.2 -c 3 -y \
--assign-identity <cluster-identity-id> \
--assign-kubelet-identity <kubelet-identity-id> \
--network-policy calico --network-plugin kubenet \
--outbound-type managedNATGateway \
-a open-service-mesh,azure-policy,azure-keyvault-secrets-provider \
--auto-upgrade-channel rapid \
--disable-local-accounts \
--no-ssh-key \
--enable-azure-rbac \
--enable-aad \
--aad-tenant-id <tenant-id> \
--aad-admin-group-object-ids <admin-group-id> \
--os-sku CBLMariner \
-s Standard_D4s_v3 \
--node-osdisk-type Ephemeral \
--node-osdisk-size 48 \
--nodepool-name base \
--nodepool-labels owner=alvozza \
--tags owner=alvozza \
--enable-pod-identity --enable-pod-identity-with-kubenet \
-g resources -n arcargo
```
If you have Pod Identity enabled, you need to add an exception:

```
kubectl create ns flux-system ; kubectl apply -f - <<EOF
apiVersion: aadpodidentity.k8s.io/v1
kind: AzurePodIdentityException
metadata:
  name: flux-extension-exception
  namespace: flux-system
spec:
  podLabels:
    app.kubernetes.io/name: flux-extension
EOF
```
Finally, deploy the extension pointing to the repo. Note the kustomization that deploys the root app is dependent on the succesful deployment of ArgoCD (as it needs its CRDs). 

```
az k8s-configuration flux create -g resources --cluster-name argoarc --name podid-arc-manifest --cluster-type managedClusters --sync-interval 30s --ns cluster-config -s cluster -u https://github.com/ams0/arc-gitops-argocd.git --branch main -k name=flux path=./flux prune=true sync_interval=30s  -k name=argocd path=./argocd/ prune=true sync_interval=30s dependsOn=flux
```

In a few minutes you'll be able to acces the ArgoCD dashboard, retrieve the admin password and use port-forwarding:

```
kubectl get secret -n argocd argocd-initial-admin-secret --template={{.data.password}} | base64 -D

kubectl port-forward svc/argocd-server 8080:80 --namespace argocd 
```

