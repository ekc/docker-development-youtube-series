# Code and script snippets

Inspiration: [Marcel's github repo](https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/hashicorp/vault-2022)

## Setup

### Setup 1/5 - pre-req in alpine + consul

``` powershell
# Create a kind cluster (1 control plan node + 3 workers)

# cd C:\dev\git\repos\ekc\docker-development-youtube-series\hashicorp\vault-2024
cd hashicorp\vault-2024
kind create cluster --config .\kind.yaml
```

``` sh
# Setup in alpine
docker run -it --rm --net host -v ${HOME}/.kube/:/root/.kube/ -v ${PWD}:/work -w /work alpine sh

# Install kubectl (alpine)

apk add --no-cache curl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl

# Install helm (alpine)

curl -LO https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz
tar -C /tmp/ -zxvf helm-v3.7.2-linux-amd64.tar.gz
rm helm-v3.7.2-linux-amd64.tar.gz
mv /tmp/linux-amd64/helm /usr/local/bin/helm
chmod +x /usr/local/bin/helm

# Optional, install vault command (alpine)

export PRODUCT=vault
export VERSION=1.17.2

apk add --update --virtual .deps --no-cache gnupg && \
    cd /tmp && \
    wget https://releases.hashicorp.com/${PRODUCT}/${VERSION}/${PRODUCT}_${VERSION}_linux_amd64.zip && \
    wget https://releases.hashicorp.com/${PRODUCT}/${VERSION}/${PRODUCT}_${VERSION}_SHA256SUMS && \
    wget https://releases.hashicorp.com/${PRODUCT}/${VERSION}/${PRODUCT}_${VERSION}_SHA256SUMS.sig && \
    wget -qO- https://www.hashicorp.com/.well-known/pgp-key.txt | gpg --import && \
    gpg --verify ${PRODUCT}_${VERSION}_SHA256SUMS.sig ${PRODUCT}_${VERSION}_SHA256SUMS && \
    grep ${PRODUCT}_${VERSION}_linux_amd64.zip ${PRODUCT}_${VERSION}_SHA256SUMS | sha256sum -c && \
    unzip /tmp/${PRODUCT}_${VERSION}_linux_amd64.zip -d /tmp && \
    mv /tmp/${PRODUCT} /usr/local/bin/${PRODUCT} && \
    rm -f /tmp/${PRODUCT}_${VERSION}_linux_amd64.zip ${PRODUCT}_${VERSION}_SHA256SUMS ${VERSION}/${PRODUCT}_${VERSION}_SHA256SUMS.sig && \
    apk del .

cd /work
# Verify kubernetes and helm readiness (alpine)

kubectl get nodes

helm repo add hashicorp https://helm.releases.hashicorp.com

# Generate Consul manifests from helm template (alpine)

helm search repo hashicorp/consul --versions

helm template consul hashicorp/consul \
  --namespace vault \
  --version 1.5.0 \
  -f consul-values.yaml \
  > ./manifests/consul.yaml

# Apply consul yaml artifacts
kubectl create ns vault
kubectl -n vault apply -f ./manifests/consul.yaml
kubectl -n vault get pods
```

### Setup 2/5 - pre-req in debian + certficate management

``` powershell
# Open a new powershell (2) - debian
# cd C:\dev\git\repos\ekc\docker-development-youtube-series\hashicorp\vault-2024\tls
cd hashicorp\vault-2024\tls

docker run -it --rm -v ${PWD}:/work -w /work debian bash
```

``` sh
# Install cfssl
apt-get update && apt-get install -y curl &&
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64 -o /usr/local/bin/cfssl && \
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64 -o /usr/local/bin/cfssljson && \
chmod +x /usr/local/bin/cfssl && \
chmod +x /usr/local/bin/cfssljson

# Review `ca-config.json` and `ca-csr.json`
# Generate certificates (+ keys) for Vault server (including its CA) (alpine)

# generate ca in /tmp
cfssl gencert -initca ca-csr.json | cfssljson -bare /tmp/ca

# generate certificate in /tmp
cfssl gencert \
  -ca=/tmp/ca.pem \
  -ca-key=/tmp/ca-key.pem \
  -config=ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ca-csr.json | cfssljson -bare /tmp/vault

ls -l /tmp
mv /tmp/* .
exit
```

### Setup 3/5 - return to alpine +  set up vault

``` sh

# Return to powershell (1) - alpine

# Create tls secret for Vault server
kubectl -n vault create secret tls tls-ca \
 --cert ./tls/ca.pem \
 --key ./tls/ca-key.pem

kubectl -n vault create secret tls tls-server \
  --cert ./tls/vault.pem \
  --key ./tls/vault-key.pem

# Generate Vault manifests from helm template

helm search repo hashicorp/vault --versions

helm template vault hashicorp/vault \
  --namespace vault \
  --version 0.28.1 \
  -f vault-values.yaml \
  > ./manifests/vault.yaml

# Deploy vault
kubectl -n vault apply -f ./manifests/vault.yaml
kubectl -n vault get pods

# Initialize and unseal vault

# Use 2/3 quorum of key-shares (alpine)

kubectl -n vault exec -it vault-0 -- vault operator init -key-shares=3 -key-threshold=2

# Paste key1 and key2 here to unseal
export key1="key1"
export key2="key2"

# valut-0
kubectl -n vault exec -it vault-0 -- vault operator unseal $key1
kubectl -n vault exec -it vault-0 -- vault operator unseal $key2
# valut-1
kubectl -n vault exec -it vault-1 -- vault operator unseal $key1
kubectl -n vault exec -it vault-1 -- vault operator unseal $key2
# valut-2
kubectl -n vault exec -it vault-2 -- vault operator unseal $key1
kubectl -n vault exec -it vault-2 -- vault operator unseal $key2

kubectl -n vault exec -it vault-0 -- vault status
kubectl -n vault exec -it vault-1 -- vault status
kubectl -n vault exec -it vault-2 -- vault status

```

### Setup 4/5 - Vault UI

``` powershell

# Open a new powershell window

kubectl -n vault get svc
kubectl -n vault port-forward svc/vault-ui 443:8200

https://localhost/
```

### Setup 5/5 - enable kubernetes authentication method

``` sh

# Return to powershell (1) - alpine

# Enable kubernetes authentication method (not kubernetes secret engine) (vault-0)

kubectl -n vault exec -it vault-0 -- sh

vault login
vault auth enable kubernetes

vault write auth/kubernetes/config \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
issuer="https://kubernetes.default.svc.cluster.local"

```

## Secret Injection Example

### Create role and policy for kubernetes authentication

``` sh

# Return to powershell (1) - alpine - vault-0
# Otherwise, kubectl -n vault exec -it vault-0 -- sh

# Define basic-secret-role
vault write auth/kubernetes/role/basic-secret-role \
   bound_service_account_names=basic-secret \
   bound_service_account_namespaces=example-app \
   policies=basic-secret-policy \
   ttl=1h

# Create a policy (vault-0)
cat <<EOF > /home/vault/app-policy.hcl
path "secret/basic-secret/*" {
  capabilities = ["read"]
}
EOF
vault policy write basic-secret-policy /home/vault/app-policy.hcl
```

### Create kv v1 secret engine and put sample data

``` sh

# Return to powershell (1) - alpine - vault-0
# Otherwise, kubectl -n vault exec -it vault-0 -- sh

# Create a basic-secret kv v1 secret engine (vault-0)

vault secrets enable -path=secret/ kv
vault kv put secret/basic-secret/helloworld username=dbuser password=sUp3rS3cUr3P@ssw0rd

# Leave vault-0
exit
```

### Deploy basic-secret secret injection application and test

``` sh
# Return to powershell (1) - alpine

# Deploy basic-secret app (to demo secret injection)
kubectl create ns example-app
kubectl -n example-app apply -f ./example-apps/basic-secret/deployment.yaml
kubectl -n example-app get pods

# Check the file with injected secret
#kubectl -n example-app exec <pod-name> -- sh -c "cat /vault/secrets/helloworld"
kubectl -n example-app exec basic-secret-########-###### -- sh -c "cat /vol01/secrets/helloworld"
```

## Vault agent example

Inspiration: [HashiCorp tutorial - Vault Agent with Kubernetes](https://developer.hashicorp.com/vault/tutorials/kubernetes/agent-kubernetes)

### Prepare file and serviceaccount

``` sh
# Return to powershell (1) - alpine

# Create a generic for ca.pem file
kubectl -n example-app create secret generic vault-ca-cert --from-file=./tls/ca.pem

# (done) kubectl create ns example-app
# Create a service account
kubectl -n example-app apply -f ./example-apps/vault-agent-example/service-account.yaml
kubectl -n example-app get serviceaccount
```

### Setup role and policy for kubernetes authentication and data for secret kv secret engine

``` sh
kubectl -n vault exec -it vault-0 -- sh

vault login
vault write auth/kubernetes/role/vault-agent-example \
     bound_service_account_names=vault-agent-example \
     bound_service_account_namespaces=example-app \
     token_policies=vault-agent-example-kv-ro \
     ttl=24h


vault policy write vault-agent-example-kv-ro - <<EOF
path "secret/vault-agent-example/*" {
    capabilities = ["read", "list"]
}
EOF

vault kv put secret/vault-agent-example/config \
      username='appuser' \
      password='suP3rsec(et!' \
      ttl='30s'

# Leave vault-0
exit
```

### Optional, verify the Kubernetes auth method configuration

``` sh
kubectl -n example-app apply -f ./example-apps/vault-agent-example/devwebapp.yaml
kubectl -n example-app get pods
kubectl -n example-app exec -it devwebapp -- sh

KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST --cacert $VAULT_CACERT \
       --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "vault-agent-example"}' \
       $VAULT_ADDR/v1/auth/kubernetes/login | python3 -m json.tool

# Leave devwebapp pod and return to the alpine container
exit
```

### Deploy and test vault-agent init container with a sample app

``` sh
kubectl -n example-app apply -f ./example-apps/vault-agent-example/configmap.yaml
kubectl -n example-app get cm,secret
kubectl -n example-app apply -f ./example-apps/vault-agent-example/vault-agent-example.yaml
kubectl -n example-app get pods

kubectl -n example-app exec -it vault-agent-example --container nginx-container -- cat /usr/share/nginx/html/index.html

# testing
kubectl -n example-app port-forward pod/vault-agent-example 8080:80
http://localhost:8080/

```

## Appendix A - Clean up

``` sh
kubectl delete ns example-app

kubectl delete -f .\manifests\vault.yaml

kubectl -n vault exec -it consul-consul-server-0 -- consul kv export
kubectl -n vault exec -it consul-consul-server-0 -- consul kv delete -recurse vault

kubectl -n vault delete -f .\manifests\consul.yaml
kubectl get pvc -A
kubectl -n vault delete pvc/data-vault-consul-consul-server-0

kubectl delete ns vault

kind get clusters
kind delete clusters kind
docker volume ls
docker system prune -a -f
```

## Appendix B - Miscellaneous Powershell

```sh
export SA_JWT_TOKEN=$(kubectl get secret $SA_SECRET_NAME \
    --output 'go-template={{ .data.token }}' | base64 -d)

export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 -d)
```

``` powershell
$env:SA_SECRET_NAME=$(kubectl get secrets --output=json `
    | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')
$env:SA_JWT_TOKEN=$(kubectl get secret $env:SA_SECRET_NAME `
    --output 'go-template={{ .data.token }}' | base64 --decode)
$env:SA_CA_CRT=$(kubectl config view --raw --minify --flatten `
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
$env:K8S_HOST=$(kubectl config view --raw --minify --flatten `
    --output 'jsonpath={.clusters[].cluster.server}')

vault auth enable kubernetes

vault write auth/kubernetes/config `
     token_reviewer_jwt="$env:SA_JWT_TOKEN" `
     kubernetes_host="$env:K8S_HOST" `
     kubernetes_ca_cert="$env:SA_CA_CRT" `
     issuer="https://kubernetes.default.svc.cluster.local"
```

ping host.docker.internal
