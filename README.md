# OpenMetadata Helm ArgoCD

Test deployment of OpenMetadata via Helm and ArgoCD 

# Table of contents

- [OpenMetadata Helm ArgoCD](#openmetadata-helm-argocd)
- [Table of contents](#table-of-contents)
- [Setup](#setup)
  - [kubectl](#kubectl)
  - [helm](#helm)
  - [kind](#kind)
  - [Storage](#storage)
  - [NGINX Ingress Operator](#nginx-ingress-operator)
  - [Cert Manager](#cert-manager)
  - [ArgoCD](#argocd)
- [OpenMetadata](#openmetadata)
  - [Install using Helm](#install-using-helm)

# Setup

## kubectl 

```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

sudo apt-get update
sudo apt-get install -y kubectl
```

## helm

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## kind

Install Docker first. Then...

```
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.26.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.26.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

To create a cluster:

```
kind create cluster --name kind
```

To delete a cluster:

```
kind delete cluster --name kind
```

List clusters:

```
kind get clusters
```

## Storage

```
kubectl apply -f deployment/dev/storage/local-storage-class.yaml 

kubectl get sc
kubectl get pv
```

## NGINX Ingress Operator

Install NGINX using helm

```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

## Cert Manager

Add Cert Manager:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.yaml
```

Add LetsEncrypt staging:

```
kubectl apply -f https://raw.githubusercontent.com/JBris/kubernetes-kind-examples/refs/heads/main/deployment/dev/certs/staging-issuer.yaml 
kubectl get issuer
```

## ArgoCD

Get CLI:

```
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

Then deploy:

```
kubectl create namespace argocd

kubectl get ns

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n argocd

# Port forwarding
kubectl port-forward -n argocd service/argocd-server 8443:443
```
Get admin password

```
argocd admin initial-password -n argocd
```

Default username for ArgoCD is admin.

Visit localhost:8443 on your web browser.

```
xdg-open localhost:8443
```

# OpenMetadata

## Install using Helm

```
kubectl create ns omd
kubectl apply -f deployment/dev/secrets/
kubectl apply -f deployment/dev/deps/

kubectl get pod -n omd
```

```
helm uninstall openmetadata-dependencies
helm uninstall openmetadata

kubectl apply -f deployment/dev/openmetadata/ 

# If needed
kubectl delete pod opensearch-0 --grace-period=0 --force --namespace default

helm install openmetadata-dependencies open-metadata/openmetadata-dependencies --values  deployment/dev/openmetadata/values-dependencies.yml

helm install openmetadata open-metadata/openmetadata
```

Cleanup:

```
kubectl delete ns omd
kubectl delete pv local-storage-elasticsearch  
kubectl delete pv local-storage-minio  
kubectl delete pv local-storage-postgres  
```