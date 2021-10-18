## Issues:

## Vault inside kubernetes

You can spawn a kind cluster as a demo as follows:
```
kind create cluster --config ./01-kind.yml
```

Allow OIDC discovery without authentication as per https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-issuer-discovery
```
kubectl create clusterrolebinding oidc-reviewer  \
  --clusterrole=system:service-account-issuer-discovery \
  --group=system:unauthenticated
```

This should make
`https://kubernetes.default.svc.cluster.local/.well-known/openid-configuration`
publicly accessible so vault can use it to configure it to trust the JWT
tokens.


And install the vault helm chart:
```
helm install vault hashicorp/vault
kubectl exec vault-0 -- vault operator init -keys-shares 1 -key-threshold -1
kubectl exec vault-0 -- vault operator unseal <key>
kubectl exec vault-0 -- vault login <root-token>

```

Then we can configure vault to use the APIServer's OIDC configuration:

```
kubectl exec vault-0 -- vault auth enable jwt
```

Set up vault to discover the issuer JWK from the kube apiserver using the discovery URI
```
kubectl exec vault-0 -- vault write auth/jwt/config \
  oidc_discovery_url=https://kubernetes.default.svc.cluster.local \
  oidc_discovery_ca_pem=@/run/secrets/kubernetes.io/serviceaccount/ca.crt
```


Lets assign the default kubernetes service account to the default vault role.
Gives access to cubbyhole and nothing else:
```
kubectl exec vault-0 -- vault write auth/jwt/role/default \
  role_type=jwt \
  bound_audiences=vault \
  user_claim=sub \
  bound_subject="system:serviceaccount:default:default" \
  policies=default ttl=1h
```

## Example with pod:

```
$ kubectl apply -f manifests/pod.yaml
$ kubectl logs vault-example
Success! Data written to: cubbyhole/foo
Key    Value
---    -----
bar    baz
```

```yaml
# manifests/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-example
spec:
  volumes:
    - name: vault-token
      projected:
        sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 7200
            audience: vault
  containers:
    - name: vault
      image: hashicorp/vault
      volumeMounts:
        - mountPath: /run/secrets/vault.hashicorp.com/serviceaccount
          name: vault-token
      env:
        - name: VAULT_ADDR
          value: http://vault:8200
      command:
        - /bin/sh
        - -c
        - |
          export VAULT_TOKEN="$(vault write -field=token auth/jwt/login role=default jwt=@/run/secrets/vault.hashicorp.com/serviceaccount/token)"
          vault write cubbyhole/foo bar=baz
          vault read cubbyhole/foo
          sleep 3000
```



## Usage with vault injector:

Vault injector can be configured as follows to use the JWT / OIDC token from kubernetes.
However there is an issue that it keeps spamming trying to delete the JWT from disk; whilst that isn't possible

vautl-agents tries to delete the JWT when consuming it https://github.com/hashicorp/vault/pull/11969  whilst kubernetes keeps the JWT as read-only and writes a new copy to it every time the token expires. Vault-agent doesn't seem to handle that scenario yet properly.


```yaml
# manifests/nginx.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-token: "true"
        vault.hashicorp.com/auth-type: "jwt"
        vault.hashicorp.com/auth-path: "auth/jwt"
        vault.hashicorp.com/auth-config-path: "/var/run/secrets/vault.hashicorp.com/serviceaccount/token"
        vault.hashicorp.com/agent-service-account-token-volume-name: "vault-token"
        vault.hashicorp.com/role: "default"
    spec:
      volumes:
        # Create a token for the default service account with vault as an audience.
        # Note that this matches up with the audience parameter in the jwt auth role config
        - name: vault-token
          projected:
            sources:
            - serviceAccountToken:
                path: token
                expirationSeconds: 7200
                audience: vault
      containers:
      - image: nginx
        name: nginx
```


