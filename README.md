

## Set up OIDC discovery in kubernetes
Allow OIDC discovery without authentication as per https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-issuer-discovery
```
kubectl create clusterrolebinding oidc-reviewer  \
  --clusterrole=system:service-account-issuer-discovery \
  --group=system:unauthenticated
```

This should make
`https://$APISERVER/.well-known/openid-configuration`
publicly accessible so vault can use it to configure it to trust the JWT
tokens.


## Vault inside kubernetes

You can spawn a kind cluster as a demo as follows:
```
kind create cluster --config ./01-kind.yml
```

And install the vault helm chart:
```
helm install vault hashicorp/vault
```

Then we can configure vault to use the APIServer's OIDC configuration:


```
vault auth enable jwt
```

Set up vault to discover the issuer JWK from the kube apiserver using the discovery URI
```
vault write auth/jwt/config \
  oidc_discovery_url=https://kubernetes.default.svc.cluster.local \
  oidc_discovery_ca_pem=@/run/secrets/kubernetes.io/serviceaccount/ca.crt
```


Lets assign the default kubernetes service account to the default vault role.
Gives access to cubbyhole and nothing else:
```
vault write auth/jwt/role/default
  role_type=jwt \
  bound_audiences=vault \
  user_claim=sub \
  bound_subject="system:serviceaccount:default:default" \
  policies=default ttl=1h
```

## Vault outside kubernetes

It is important that the issuer in your service account tokens matches the external URL of your apiserver. The OIDC spec mandates that these fields are in sync. This requires setting the `service-account-issuer` flag to the external URL of the apiserver in the apiserver config.

An example of this is in [./kind/02-kind-pinned.yml](./kind/02-kind-pinned.yml).
```
kind create cluster --config ./kind/02-kind-pinned.yml
```

Then we can set up an external vault cluster in a separate terminal:
```
vault server -dev
```

Then you can configure vault as follows:

```
APISERVER_URL=https://127.0.0.1:6443 # or your external url
vault write auth/jwt/config \
  oidc_discovery_url=$APISERVER_URL \
  oidc_discovery_ca_pem=$(kubectl get configmap kube-root-ca.crt -ojsonpath="{.data['ca\.crt']}")
```

The role config stays the same:



## Usage with vault injector:

Vault injector can be configured as follows to use the JWT / OIDC token from kubernetes:

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
