# Initialisation de vault grace a Helm dans kubernetes

```sh
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault -n vault --create-namespace
```

## Initialisation de vault par cle de recuperation
```sh
kubectl exec -it vault-0 -n vault -- vault operator init

```

## Activation du moteur database
```sh
vault secrets enable database
```

## Configuration du vault et stockage des identifiant postgres
```sh
vault write database/config/postgres \
    plugin_name=postgresql-database-plugin \
    allowed_roles="readonly" \
    connection_url="postgresql://{{username}}:{{password}}@pg-cluster-primary:5432/postgres" \
    username="admin" \
    password="SuperSecurePassword"
```

## Definition de rotation automatique de mot de passe
```sh
vault write database/roles/readonly \
    db_name=postgres \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
    default_ttl="1h" \
    max_ttl="24h"
```


# Configuration de vault dans kubernetes
```sh
vault auth enable kubernetes
```

## Configuration de l'authentification kubernetes
```sh
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
```

## Creation de secret dynamique avec vault
```sh
vault write auth/kubernetes/role/postgres-role \
    bound_service_account_names=pgadmin \
    bound_service_account_namespaces=default \
    policies=postgres-policy \
    ttl=1h

```
## Creation de policy vault
```sh
vault policy write postgres-policy - <<EOF
path "database/creds/readonly" {
  capabilities = ["read"]
}
EOF
```

## Creation du vault dynamique secret:
```sh
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultDynamicSecret
metadata:
  name: postgres-secret
spec:
  mount: database
  path: creds/readonly
  namespace: default
  ttl: 1h

```


## Audit vault pour tracer les requetes
```sh
vault audit enable file file_path=/vault/logs/audit.log

```

# Test en local de vault
Lancer le vault en mode dev
```sh
vault server -dev
```
