# Overview

This project represents an Application or Package State Repository used by Consumer Edge and
ConfigSync on GCP. The purpose of this repository is to contain the KRM manifest files to
run a Point of Sales application at an edge location.

# Steps to Use

1. Have a cluster running with ConfigSync enabled
1. Have a cluster with External Secrets installed and configured to your GCP project (suggest appling the Cluster Trait Repo: https://gitlab.com/gcp-solutions-public/retail-edge/available-cluster-traits/external-secrets-anthos )
1. Create a Personal Access Token for your SCM provider
1. Use a Primary Root Repository for your edge cluster
1. Setup the Configuration (see /config-manifests) in the namespace of the Package State Repository config files (see below)
1. Add the Configuration for the repository (sample below)

# Creating GCP Secret

```shell
export PROJECT_ID="<your gcp project id>"
export SCM_TOKEN_USER="<personal access token user>"
export SCM_TOKEN_TOKEN="<personal access token value>"

gcloud secrets create pos-git-token-creds --replication-policy="automatic" --project="${PROJECT_ID}"
echo -n "{\"token\"{{':'}} \"${SCM_TOKEN_TOKEN}\", \"username\"{{':'}} \"${SCM_TOKEN_USER}\"}" | gcloud secrets versions add pos-git-token-creds --project="${PROJECT_ID}" --data-file=-
```

# Sample RepoSync

```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RepoSync
metadata:
  name: pos-rs                                 # 6 characters in length
  namespace: loc-config
spec:
  sourceFormat: "unstructured"
  git:
    repo: "https://gitlab.com/gcp-solutions-public/retail-edge/gdc-demos/point-of-sales.git"
    branch: "main"
    dir: "/manifests"
    auth: "token"
    secretRef:
      name: pos-git-creds                     # K8s secret produced by ExternalSecret below

---

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: pos-git-creds-es
  namespace: pos-app
spec:
  refreshInterval: 24h
  secretStoreRef:
    kind: ClusterSecretStore
    name: gcp-secret-store
  target:                                       # K8s secret name
    name: pos-git-creds                         # Matches the secretRef above
    creationPolicy: Owner
  data:
    - secretKey: username                       # K8s secret key name to be set inside Secret
      remoteRef:
        key: pos-git-token-creds                # GCP Secret Name (https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets)
        property: username                      # field inside GCP Secret
    - secretKey: token                          # K8s secret key name to be set inside Secret
      remoteRef:
        key: pos-git-token-creds                # GCP Secret Name (https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets)
        property: token                         # field inside GCP Secret

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pos-package-repo
  namespace: pos-app
subjects:
  - kind: ServiceAccount
    name: ns-reconciler-pos-app-pos-rs-6        # k get sa -n config-management-system  ( ns-reconciler-{NAMESPACE}-{REPO_SYNC_NAME}-{REPO_SYNC_NAME_LENGTH} )
    namespace: config-management-system
roleRef:
  kind: ClusterRole
  name: admin                                   # Granular control over resources can be created with RBAC (Future demo will explore custom RBAC)
  apiGroup: rbac.authorization.k8s.io

```