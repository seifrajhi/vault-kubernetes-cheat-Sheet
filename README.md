# vault-kubernetes-cheat-Sheet: A full cheat sheat for vault for kubernetes secrets

This guide will walk you through the process of setting up HashiCorp Vault to manage Kubernetes secrets using Helm.

## Prerequisites

- Kubernetes cluster
- Helm v3
- `kubectl` command-line tool

## Steps

1. Add the HashiCorp Helm repository:

    ```bash
    helm repo add hashicorp https://helm.releases.hashicorp.com
    ```

2. Update the Helm repository:

    ```bash
    helm repo update
    ```

3. Install the Vault Helm chart:

    ```bash
    helm install vault hashicorp/vault
    ```

4. Initiate and unseal vault  

    ```bash
    kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > keys.json

    VAULT_UNSEAL_KEY=$(cat keys.json | jq -r ".unseal_keys_b64[]")
    echo $VAULT_UNSEAL_KEY

    VAULT_ROOT_KEY=$(cat keys.json | jq -r ".root_token")
    echo $VAULT_ROOT_KEY

    kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
  
    kubectl exec vault-0 -- vault login $VAULT_ROOT_KEY
  
    
    kubectl get pods

    ```




5. Create a service account for Vault:

    ```bash
    kubectl create serviceaccount vault-auth
    ```


6. Log in to the Vault pod:

    ```bash
    kubectl exec -it vault-0 -- /bin/sh
    ```

7. Enable the Kubernetes authentication method in Vault:

    ```bash
    vault auth enable kubernetes
    ```

8. Configure the Kubernetes authentication method in Vault:

    ```bash
    vault write auth/kubernetes/config \
      token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
      kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT_HTTPS}" \
      kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    ```

9. Create a policy for your application:

    ```bash
    cat <<EOF > myapp-policy.hcl
    path "secret/myapp/access" {
      capabilities = ["create", "update", "read"]
    }
    EOF
    ```

10. Create the policy in Vault:

    ```bash
    vault policy write myapp-policy myapp-policy.hcl
    
    ## second option
    vault policy write myapp-policy - << EOF
    path "secret/myapp/access" {
        capabilities = ["create", "update", "read"]
    }
    EOF
    ```

12. Create a Kubernetes role for Vault:

    ```bash
    vault write auth/kubernetes/role/vault-aws \
      bound_service_account_names=vault-auth \
      bound_service_account_namespaces=default \
      policies=myapp-policy \
      ttl=24h
    ```

13. Create a Kubernetes secret in Vault:

    ```bash
    vault kv put secret/myapp/access username="myuser" password="mypassword"
    ```

14. Verify that the Kubernetes secret was created in Vault:

    ```bash
    vault kv get secret/myapp/access
    ```

Note that you may need to modify some of the values (e.g., the `bound_service_account_names` in Step 12) to fit your specific use case. Also, make sure to replace `myapp` with the name of your own application.



15. Inject the secret into nginx pod

You can use annotations in your pod's YAML file to specify which secrets to inject and how to inject them. Here's an example YAML file:
```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-username: creds/vault/access
        vault.hashicorp.com/role: vault-aws
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
      securityContext: {}
      serviceAccount: vault-auth 
      serviceAccountName: vault-auth 
```
