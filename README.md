# arc-gitops-argocd
Use ArgoCD with Arc-enabled Kubernetes clusters

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


```
az k8s-configuration flux create -g resources --cluster-name argoarc --name podid-arc-manifest --cluster-type managedClusters --sync-interval 30s --ns cluster-config -s cluster -u https://github.com/ams0/arc-gitops-argocd.git --branch main -k name=flux path=./flux prune=true sync_interval=30s  -k name=argocd path=./argocd/ prune=true sync_interval=30s dependsOn=flux
```