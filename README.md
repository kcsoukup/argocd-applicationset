# ArgoCD ApplicationSet using a Git Generator to Deploy Apps
This is a quick and dirty guide to using ArgoCD ApplicationSets to iterate through a Git Repository and install multiple applications dynamically, a GitOps pattern.

Note: The namespace generator is reliant on https://github.com/kcsoukup/k8-namespace-provisioning for Helm dependencies... another area of study.


#### Repo Directory Structure
Each namespace has its own Helm Charts, which uses a dependency Helm Chart for consistency
```
.
├── applicationset.yaml
├── namespaces
│   ├── megadeth
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── mushroomhead
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── pantera
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   ├── README.md
│   └── slayer
│       ├── Chart.yaml
│       └── values.yaml
└── README.md
```

#### Namespace Values File
To add new namespace: duplicate other namespace directory, change directory name and then update values.yaml namespace.name.
```
k8-namespace-provisioning:
  namespace:
    name: mushroomhead
```

#### ApplicationSet Manifest

Contents of applicationset.yaml
```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-namespaces
spec:
  generators:
    - git:
        repoURL: https://github.com/kcsoukup/argocd-applicationset.git
        revision: HEAD
        directories:
          - path: "namespaces/*"
  template:
    metadata:
      name: '{{path.basename}}-appset-namespace'
      annotations:
        argocd.argoproj.io/sync-wave: "-1"
    spec:
      project: default
      source:
        repoURL: https://github.com/kcsoukup/argocd-applicationset.git
        targetRevision: HEAD
        path: '{{path}}'
        helm:
          valueFiles:
            - values.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

```

Apply manifest

`kubectl apply  -f applicationset.yaml -n argocd`

#### Check Points
ArgoCD should show the Namespace apps within 3mins

Check Kubernetes

`kubectl get namespaces`

```
NAME                   STATUS   AGE
argocd                 Active   6d6h
damage-inc             Active   17d
default                Active   18d
ingress-nginx          Active   17d
kube-flannel           Active   18d
kube-node-lease        Active   18d
kube-public            Active   18d
kube-system            Active   18d
kubernetes-dashboard   Active   14d
megadeth               Active   21m  <-- Worked!
metallb-system         Active   17d
mushroomhead           Active   21m  <-- Worked!
pantera                Active   16m  <-- Worked!
slayer                 Active   16m  <-- Worked!
```

...or...

`argocd app list`

```
NAME                                  CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                             PATH                     TARGET
argocd/appset-namespace-megadeth      https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>      https://github.com/kcsoukup/argo-appsofapps.git  namespaces/megadeth      HEAD
argocd/appset-namespace-mushroomhead  https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>      https://github.com/kcsoukup/argo-appsofapps.git  namespaces/mushroomhead  HEAD
argocd/appset-namespace-pantera       https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>      https://github.com/kcsoukup/argo-appsofapps.git  namespaces/pantera       HEAD
argocd/appset-namespace-slayer        https://kubernetes.default.svc  argocd     default  Synced  Healthy  Auto-Prune  <none>      https://github.com/kcsoukup/argo-appsofapps.git  namespaces/slayer        HEAD
```
