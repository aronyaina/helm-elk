# Installation
Meilleur moyen d'installation utilisant l'operateur crunchy data

```sh
git clone https://github.com/CrunchyData/postgres-operator-examples.git
```

## Conservation seulement des dossiers utilisés dans un nouveau dossier
`new_dir` référence le répertoire sur lequel nous allons organiser notre nouveau dossier.

```sh
cp -r install new_dir
cp -r postgres new_dir
cp -r pgadmin new_dir
```

Installer l'opérateur avec Kustomize Kubernetes.
```sh
kubectl apply -k new_dir/install --server-side --force-conflicts
```

# Configuration de postgres default user et password
## Creation du mot de passe crunchyData
```yml
apiVersion: v1
kind: Secret
metadata:
  name: pg-user-secret
type: Opaque
stringData:
  password: "MonMotDePasseSecurisé"
---
apiVersion: v1
kind: Secret
metadata:
  name: pg-user1-secret
type: Opaque
stringData:
  password: "MotDePasseUser1"
---
apiVersion: v1
kind: Secret
metadata:
  name: pg-user2-secret
type: Opaque
stringData:
  password: "MotDePasseUser2"
```

## Reference du mot de passe securise

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: crunchy-pgcluster
spec:
  postgresVersion: 17
  users:
    - name: user
      databases:
        - test_db
      options: SUPERUSER
      password:
        name: pg-user-secret  
    - name: user1
      databases:
        - test_db
      options: SUPERUSER
      password:
        name: pg-user1-secret  
    - name: user2
      databases:
        - test_db
      options: CREATEDB
      password:
        name: pg-user2-secret 
  instances:
    - name: pgha
      replicas: 3
      dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 1Gi
        storageClassName: microk8s-hostpath
  backups:
    pgbackrest:
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 1Gi
            storageClassName: microk8s-hostpath
```

## Meilleur pratique de deploiement pour crunchydata
High Availability : Disponible a partir de 3 replicas
Augmentation de 50Go pour la taille du stockage en prod avec les stockage en ligne
```yml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: pg-cluster
spec:
  postgresVersion: 17
  instances:
    - name: pgha
      replicas: 3  # ✅ Minimum 3 réplicas pour HA
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 10Gi  # ✅ Augmenter la taille du stockage en prod
        storageClassName: fast-storage
```

# Monitoring requis dans postgres
Utilisation de Postgres Metric dans grafana : [PostgreSQL Metrics]
```yml
  monitoring:
    pgmonitor:
      enabled: true
```

PGADMIN ---
pgadmin deja integrer avec l'operateur, modification de mot de passe comme tout a l'heureavec des secrets
```yml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PGAdmin
metadata:
  name: rhino
spec:
  users:
    - username: rhino@example.com
      role: Administrator
      passwordRef:
        name: pgadmin-password-secret
        key: rhino-password
  dataVolumeClaimSpec:
    accessModes:
    - "ReadWriteOnce"
    resources:
      requests:
        storage: 1Gi
    storageClassName: microk8s-hostpath
  serverGroups:
    - name: supply
      postgresClusterSelector: {}
  config:
    settings:
      AUTHENTICATION_SOURCES: ['internal']
```

# Backup Complete
1. En Ligne
```yml
  backups:
    pgbackrest:
      repos:
        - name: repo1
          s3:
            bucket: my-postgres-backups
            endpoint: s3.amazonaws.com
            region: us-east-1
          retention:
            full: 7  
```
2. En Local
```yml
  backups:
    pgbackrest:
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 1Gi
            storageClassName: microk8s-hostpath
```

# Mise a jour de postgres
Rolling Update, Mise a jour du replicas puis le leader
Test de mise a jour dans le staging avant le prod

# Restriction des acces avec les network policies
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pg-cluster-policy
spec:
  podSelector:
    matchLabels:
      postgres-operator.crunchydata.com/cluster: pg-cluster
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: my-app
```
