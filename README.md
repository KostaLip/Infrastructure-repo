# Infrastructure Repo: GitOps Manifests

This is the **GitOps repository** of the [Cloud-Native CI/CD Pipeline](https://gitlab.com/kostalip/microservices-gitops) project. It holds the Kubernetes manifests for the [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) microservices application and acts as the single source of truth for what runs on the local Kubernetes cluster.

The application code, the Terraform infrastructure and the CI/CD pipeline live in the [GitLab repository](https://gitlab.com/kostalip/microservices-gitops). The pipeline there updates the manifests in this repository, and **Argo CD** applies the changes to the cluster.

## Repository structure

```
.
├── application.yaml   # Argo CD Application manifest
└── k8s/               # Kubernetes manifests
    ├── ...            # Deployments, Services, ConfigMap, PV and PVC
```

- **`application.yaml`** defines the Argo CD `Application`. It tells Argo CD which repository, branch and path to watch and which cluster and namespace to deploy to.
- **`k8s/`** contains the Deployment and Service for every microservice, the shared ConfigMap, and the storage resources for Redis.

## Argo CD Application

The `Application` watches the `k8s/` folder on the `main` branch of this repository and deploys its contents to the local cluster.

| Setting | Value | Purpose |
|---|---|---|
| `source.path` | `k8s` | Folder that contains all manifests |
| `source.directory.recurse` | `true` | Manifests can be organized into subfolders |
| `syncPolicy.automated` | enabled | Changes in Git are applied automatically, without a manual sync |
| `prune` | `true` | Resources removed from Git are also removed from the cluster |
| `selfHeal` | `true` | Manual changes in the cluster are reverted to match Git |
| `finalizers` | `resources-finalizer.argocd.argoproj.io` | Deleting the Application also deletes the resources it created |

## How it works

1. The CI/CD pipeline in GitLab builds new images and updates the manifests in this repository.
2. Argo CD, running on the local minikube cluster, detects the change in Git.
3. Argo CD synchronizes the cluster so that it matches the manifests.

Nothing is applied to the cluster manually. Git is the desired state, and thanks to automated sync with pruning and self-healing, Argo CD keeps the cluster aligned with it, including reverting any manual drift.

## Repository access

Argo CD reads this repository using a **Kubernetes Secret** created locally in the `argocd` namespace. The Secret holds an access token with permission to read the repository, and it is registered with Argo CD as repository credentials. The Secret is created manually on the cluster and is **not stored in Git**, so no credentials are exposed in the repository.

## Design decisions

**Networking.** Every microservice is exposed with a `ClusterIP` Service, so services are reachable only from inside the cluster. The only exception is the `frontend`, which uses a `NodePort` Service so the application can be accessed from outside the cluster.

**Configuration.** Configuration is kept in a **ConfigMap**. The microservices read it as environment variables, which keeps configuration separate from the container images and in one place.

**Storage.** Redis uses a **PersistentVolume** and a **PersistentVolumeClaim**, so the data survives pod restarts. The cluster has a single node and is meant as a local test environment, so the volume is backed by a node-local path (`hostPath`).

## Related

- Main project (application, Terraform, pipeline): <https://gitlab.com/kostalip/microservices-gitops>
- Original application: [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) (Apache License 2.0)
