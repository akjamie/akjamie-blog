---
layout: post
title:       "Vault Agent with Kubernetes"
description: "Nearly all requests to Vault must be accompanied by an authentication token. This includes all API requests,
as well as via the Vault CLI and other libraries, therefore application running in kubernetes is no exception. Luckily,
Vault provides Kubernetes auth method to authenticate the clients using a Kubernetes Service Account Token, and Vault
Agent which could be leveraged to automatically inject the secrets from vault into kubernetes pods through init container
pattern."
date:        "2023-03-26"
author:      "Jamie"
image:       "/img/background-01.jpg"
tags:        ["Vault", "Kubernetes"]
categories:  ["Cloud" ]
---
# Challenge
Nearly all requests to Vault must be accompanied by an authentication token. This includes all API requests, as well as
via the Vault CLI and other libraries. If you can securely get the first secret from an originator to a consumer,
all subsequent secrets transmitted between this originator and consumer can be authenticated with the trust
established by the successful distribution and user of that first secret.

The applications running in a Kubernetes environment is no exception. Luckily, Vault provides Kubernetes auth method to
authenticate the clients using a Kubernetes Service Account Token.

However, Client is still responsible for managing the lifecycle of its Vault Tokens as illustrated on below diagram of
Vault Workflow on kubernetes.
<img src='/img/2023-03-26-vault-agent-with-k8s/vault-k8s-workflow-01.png' style="height: 840px;margin-left: 0px;"/>
```markdown
Major processes:
 - One-off setup/infrequent action for initial configuration & standard policy maintenance, e.g. enable auth method,  
create roles,policies.
 - Pod deployed with JWT token(service account token) injected.
 - API calls to Vault to authenticate from pod, Vault verifies the token and determine policies attached to the Auth token,
and return to client.
 - API calls to Vault with Auth token passed to retrieves required secrets.
```

# Vault Agent solution
Vault agent provides helper features to address the following challenges:
- Automatic authentication  
- Secure delivery/storage of tokens  
- Lifecycle management of these tokens (renewal & re-authentication)  
  <img src='/img/2023-03-26-vault-agent-with-k8s/vault-agent-01.png' style="height: 220px;margin-left: 0px;"/>

# Configure kubernetes authentication
Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account
Token. This token is provided to each pod when it is created.
## Create Vault service account
1. Create k8s manifest file
```shell
export WORKDIR=$(pwd)

cat > ${WORKDIR}/vault-auth-service-account.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
EOF
```
2. Create the vault-auth service account.
Important:  specify a namespace here!!!
```shell
kubectl apply --filename vault-auth-service-account.yaml -n default
```
3. (Kubernetes 1.24+ only) Create the secret explicitly
```shell
cat > ${WORKDIR}/vault-auth-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
EOF

kubectl apply --filename vault-auth-secret.yaml
```

## Configure Kubernetes auth method
1. Configure vault url & login with root token.
```shell
# service url is also fine, here i changed master's host, and set ingress controller's IP to vault.it-meta.space also.
export VAULT_ADDR=https://vault.it-meta.space
export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")
vault login token=$CLUSTER_ROOT_TOKEN
```
Sample output:
```shell
root@master:/opt/k8s/config-repo/vault# export VAULT_ADDR=https://vault.it-meta.space
export CLUSTER_ROOT_TOKEN=$(cat ${WORKDIR}/cluster-keys.json | jq -r ".root_token")
vault login token=$CLUSTER_ROOT_TOKEN
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.oemGqKix23q5FMi6qkXiLsqN
token_accessor       BVDmPTEBLgdxLEvQs4p5bXaD
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```
2. Set a secret in Vault
```shell
vault secrets enable -path=secret kv
vault kv put secret/data/app/config username="jamie" password="HLWD20230326"
vault kv get secret/data/app/config
```

3. Configure Kubernetes authentication method
```shell
export SA_SECRET_NAME=$(kubectl get secrets --output=json \
    | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')
export SA_JWT_TOKEN=$(kubectl get secret $SA_SECRET_NAME \
    --output 'go-template={{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
export K8S_HOST=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.server}')
    
vault auth enable kubernetes

vault write auth/kubernetes/config \
     token_reviewer_jwt="$SA_JWT_TOKEN" \
     kubernetes_host="$K8S_HOST" \
     kubernetes_ca_cert="$SA_CA_CRT" \
     issuer="https://kubernetes.default.svc.cluster.local"
```

4. Create admin & devops roles
```shell
vault write auth/kubernetes/role/app-admin \
     bound_service_account_names=vault-auth \
     bound_service_account_namespaces=default \
     token_policies=policy-app-kv-rw \
     ttl=1h

vault write auth/kubernetes/role/app-devops \
     bound_service_account_names=vault-auth \
     bound_service_account_namespaces=default \
     token_policies=policy-app-kv-ro \
     ttl=4h   
```
5. Create RW & RO policies
```shell
vault policy write policy-app-kv-rw - <<EOF
path "secret/data/app/*" {
    capabilities = ["read", "list", "delete", "create", "update"]
}
EOF

vault policy write policy-app-kv-ro - <<EOF
path "secret/data/app/*" {
    capabilities = ["read", "list"]
}
EOF
```

# Lunch application for testing
1. Create deployment/pod
```shell
cat > ${WORKDIR}/test-deployment-with-vault-agent.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: test-vault-agent
spec:
  selector:
    matchLabels:
      app: test-vault-agent
  replicas: 1
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'app-devops'
        vault.hashicorp.com/tls-skip-verify: "true"
        vault.hashicorp.com/agent-pre-populate-only: 'true'
        vault.hashicorp.com/agent-inject-secret-app-config.yaml: 'secret/data/app/config'
      labels:
        app: test-vault-agent
    spec:
      serviceAccountName: vault-auth
      containers:
        - name: nginx
          image: nginx:latest
          imagePullPolicy: IfNotPresent
EOF

k apply -f test-deployment-with-vault-agent.yaml
```
2. Login pod to check the secrets
```shell
root@master:/opt/k8s/config-repo/vault# kubectl exec $(kubectl get pod -l app=test-vault-agent -o jsonpath="{.items[0].metadata.name}") --container nginx -- cat /vault/secrets/app-config.yaml
password: HLWD20230326
username: jamie
```
3. Update secrets in vault, it seems cannot sync to pod automatically,  a quick apply way is to scale the app to 0 and scale back to original replicas.

# Integrate with Spring Cloud Vault.
Continue!

Reference docs:  
[https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar)
