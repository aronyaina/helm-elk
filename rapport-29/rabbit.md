# Installation

Meilleur moyen d’installer RabbitMQ est depuis l’operateur rabbit mq

### Securite

Implementation de mot de passe et utilisateur avec les secret

```bash
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq
  namespace: rabbit
spec:
  replicas: 3
  service:
    type: ClusterIP
  persistence:
    storageClass: "microk8s-hostpath"
    size: 5Gi
  configuration: |
    loopback_users.guest = false
    listeners.tcp.default = 5672
    cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
    cluster_formation.k8s.address_type = hostname
    cluster_formation.k8s.service_name = rabbitmq-nodes
  users:
    - name: $(RABBITMQ_USER)
      password: $(RABBITMQ_PASSWORD)
      tags: administrator
  env:
  - name: RABBITMQ_PASSWORD
    valueFrom:
      secretKeyRef:
        name: rabbitmq-password
        key: password
  - name: RABBITMQ_USER
    valueFrom:
      secretKeyRef:
        name: rabbitmq-user
        key: user
```

# Bonne pratique et test

### RABBITMQ

### BUG I (Verification de l’erlang Cookies)

Verifier que les erlang cookie sont les meme pour chaque instance avec la commande

```bash
 kubectl exec definition-server-0 -- cat /var/lib/rabbitmq/.erlang.cookie\nkubectl exec definition-server-1 -- cat /var/lib/rabbitmq/.erlang.cookie
```

### BUG II (node Qui se join pas au cluster)

Verifier dans le dashboard de base pour s’assurer que les nodes peuve se voir entre eux

Bug lieer avec microk8s jusque la

### Version stable a priorise suivant l’operateur utilise
Version utilisable de la meme version majeur seulemnt avec 4.x.x


### Bonne Pratique
- Limiter les utilisateurs specifique , seulement les utilisateurs utiliser a part l’administrateur
- Activation du TLS pour securise entre le client et le serveur

## Minimum 10Go a 20 Go et augmenter a l’utilisation
10-20 Go pour des charges légères.
50-100 Go pour des charges moyennes à lourdes

Chaque volume doit avoir une espace dedier , PVC unique , Jamais rassembler en une seul PVC

### Activation de liveness prob et readiness prob (active de base sur l’operator)

### Gestion de temps de reponse

Limiter la persistance de message , utiliser TTL pour l’expiration


# Observabilite -- Quels sont les metrics
 - Métriques du Clustering et des Nœuds
erlang_vm_memory_total	Mémoire totale utilisée par RabbitMQ.
erlang_vm_processes_used	Nombre de processus Erlang actifs.
erlang_vm_schedulers_online	Nombre de cœurs CPU utilisés par RabbitMQ.
rabbitmq_running	1 si le nœud RabbitMQ est actif, 0 sinon.
rabbitmq_cluster_nodes	Nombre de nœuds connectés au cluster.

 - Métriques de Stockage et d’I/O
Métrique	Description
disk_free_bytes	Espace disque libre sur le nœud RabbitMQ.
queue_disk_writes_total	Nombre total d'écritures disque effectuées.
queue_disk_reads_total	Nombre total de lectures disque effectuées.


# Upgrade
Rolling Update (sans downtime, si bien géré)

    Chaque nœud est mis à jour un par un sans interrompre le service.

    Fonctionne si l’upgrade est compatible entre versions.

    Requiert que les messages et connexions soient bien répliqués entre les nœuds.

Blue-Green Deployment (nécessite plus de ressources, mais sûr)

    Un nouveau cluster RabbitMQ est déployé avec la nouvelle version.

    Une fois prêt, le trafic est redirigé vers le nouveau cluster.

    Évite les risques liés aux migrations, mais double temporairement l’utilisation des ressources.
Verifier si l'upgrade peut etre fait directement
    Verification de la version ex: v3.9 - v3.11 (requis passage vers v3.10)


Gestion na ressource
```yml
resources:
  requests:
    memory: "512Mi"  # RAM minimale allouée
    cpu: "250m"       # CPU minimum requis
  limits:
    memory: "2Gi"     # RAM maximale avant redémarrage
    cpu: "1"          # CPU max utilisé
```

# Backup 
Automatisation du backup dans kubernetes
## Sauvegarde entiere
1. Utilisation de Velero (Avec un stockage a distance et S3) [Automatic]
```sh
velero install \
  --provider aws \
  --bucket my-backup-bucket \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=us-east-1
```

2. Sauvegarde rapide sans velero [Manuel]
```sh
# Copie du backup
kubectl cp rabbitmq-0:/var/lib/rabbitmq ./rabbitmq-backup -n rabbitmq
# Restauration
```

3. Cronjob [local]
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rabbitmq-backup
  namespace: rabbitmq
spec:
  schedule: "0 2 * * *"  # Exécute le backup tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              cp -r /var/lib/rabbitmq /backup/
              echo "Backup terminé à $(date)"
            volumeMounts:
            - name: rabbitmq-data
              mountPath: /var/lib/rabbitmq
            - name: backup-storage
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: rabbitmq-data
            persistentVolumeClaim:
              claimName: rabbitmq-pvc  # PVC de RabbitMQ
          - name: backup-storage
            persistentVolumeClaim:
              claimName: rabbitmq-backup-pvc  # PVC pour stocker le backup
```

## Sauvegarde partielle
Export des configurations et messages (rabbitmqctl et rabbitmq-dump-queue)	Restauration rapide, fichier JSON léger	Ne sauvegarde pas les fichiers physiques
```sh
kubectl exec -it rabbitmq-0 -n rabbitmq -- rabbitmqctl export_definitions /var/lib/rabbitmq/backup.json
kubectl cp rabbitmq-0:/var/lib/rabbitmq/backup.json ./backup.json -n rabbitmq
# Restauration
kubectl cp ./backup.json rabbitmq-0:/var/lib/rabbitmq/backup.json -n rabbitmq
kubectl exec -it rabbitmq-0 -n rabbitmq -- rabbitmqctl import_definitions /var/lib/rabbitmq/backup.json

# Sauvegarde
rabbitmq-dump-queue -h rabbitmq -u user -p password -q ma_file > messages.json
# Restauration
rabbitmq-publish -h rabbitmq -u user -p password -q ma_file -f messages.json
```

## Cronjob [Distant]
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rabbitmq-backup-s3
  namespace: rabbitmq
spec:
  schedule: "0 2 * * *"  # Exécute le backup tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: amazon/aws-cli
            command:
            - /bin/sh
            - -c
            - |
              tar -czf /tmp/rabbitmq-backup.tar.gz /var/lib/rabbitmq
              aws s3 cp /tmp/rabbitmq-backup.tar.gz s3://mon-bucket-rabbitmq/backups/rabbitmq-backup-$(date +%F).tar.gz
            env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: accessKey
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secretKey
            volumeMounts:
            - name: rabbitmq-data
              mountPath: /var/lib/rabbitmq
          restartPolicy: OnFailure
          volumes:
          - name: rabbitmq-data
            persistentVolumeClaim:
              claimName: rabbitmq-pvc
```
