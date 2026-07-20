# Otus_DevOps_Advanced_GitOps_Apps
Install Vault

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  --set "server.dev.enabled=true"

kubectl port-forward svc/vault -n vault 8200:8200

helm upgrade -i argocd argo-cd/argo-cd -n argocd --create-namespace -f values.yaml -f values-avp.yaml


vault login -address=http://127.0.0.1:8200 -tls-skip-verify
vault kv put -address=http://127.0.0.1:8200 --tls-skip-verify secret/app username=admin password=cGFzc3dvcmQ=
vault auth enable -address=http://127.0.0.1:8200 --tls-skip-verify kubernetes

cat <<EOF > policy.hcl
# policy.hcl
path "secret/*" {
  capabilities = ["read"]
}
EOF

vault policy write -address=http://127.0.0.1:8200 --tls-skip-verify argocd-policy policy.hcl

SA_NAME=argocd-repo-server
NAMESPACE=argocd
kubectl create token argocd-repo-server -n argocd

TOKEN_REVIEW_JWT=$(kubectl create token argocd-repo-server -n argocd)
KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten -o jsonpath="{.clusters[0].cluster.certificate-authority-data}" | base64 -d)
KUBE_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath="{.clusters[0].cluster.server}")

vault write -address=http://127.0.0.1:8200 -tls-skip-verify auth/kubernetes/config \
  token_reviewer_jwt="$VAULT_TOKEN" \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert="$KUBE_CA_CERT"

vault write -address=http://127.0.0.1:8200 -tls-skip-verify auth/kubernetes/role/argocd-role \
    bound_service_account_names=argocd-repo-server \
    bound_service_account_namespaces=argocd \
    policies=argocd-policy \
    ttl=48h

k apply -f avp-secret.yaml

## Generator Beispiel
Beispiel: ApplicationSet mit Git Generator
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-appset
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/org/infrastructure-repo.git
        revision: HEAD
        directories:
          - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
      labels:
        app: '{{path.basename}}'
        managed-by: appset
    spec:
      project: default
      source:
        repoURL: https://github.com/org/infrastructure-repo.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
Angenommen in infrastructure-repo existiert:

apps/
├── frontend/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── backend/
│   └── ...
└── redis/
    └── ...
    
Der Generator erzeugt automatisch 3 Applications: frontend, backend, redis – jede deployed aus dem jeweiligen Verzeichnis in ihren eigenen Namespace.

Weitere Git-Generator-Varianten:

# Branch Generator: eine App pro Branch
generators:
  - git:
      repoURL: https://github.com/org/app-repo.git
      revision: HEAD
      files:
        - path: "config.json"
        
# File Generator: eine App pro gefundener Konfig-Datei
generators:
  - git:
      repoURL: https://github.com/org/app-repo.git
      revision: HEAD
      files:
        - path: "environments/*/config.yaml"
