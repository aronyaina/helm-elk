# Installation de keycloak
Installation de keycloak avec helm et le values.yaml
## Securisation de keycloak
1. Creation de secret
```yml
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-db-secret
type: Opaque
data:
  username: Ym5fa2V5Y2xvYWs=  # base64("bn_keycloak")
  password: c2VjdXJlUGFzc3dvcmQ=  # base64("securePassword123")
  database: Yml0bmFtaV9rZXljbG9haw==  # base64("bitnami_keycloak")

---
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-admin-secret
type: Opaque
data:
  admin-user: YWRtaW4=  # base64("admin")
  admin-password: c2VjdXJlQWRtaW4h  # base64("secureAdminPassword!")
```

2. Appel des secrets dans keycloak
```yml
global:
  postgresql:
    auth:
      existingSecret: keycloak-db-secret  # Utilise le secret pour l'auth DB

auth:
  existingSecret: keycloak-admin-secret  # Utilise le secret pour l'admin Keycloak

postgresql:
  enabled: true
  primary:
    persistence:
      enabled: true
      storageClass: "microk8s-hostpath"
      size: 8Gi
      accessModes:
        - ReadWriteOnce
  auth:
    existingSecret: keycloak-db-secret  # Référence le même secret pour Postgres

extraEnvVars:
  - name: KC_PRODUCTION
    value: "true"
  - name: KC_HOSTNAME_STRICT
    value: "false"
  - name: KC_HTTP_ENABLED # Permet la connexion non securise
    value: "false"
  - name: VERTX_WORKER_POOL_SIZE
    value: "20"
  - name: VERTX_EVENT_LOOP_POOL_SIZE
    value: "8"
  - name: KC_PROXY # Derriere un proxy
    value: "edge" 
  - name: KC_METRICS_ENABLED # Metric pour grafana et prometeus
    value: "true"
  - name: KC_LOG_LEVEL
    value: "INFO"
```
