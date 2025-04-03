# Installation de Helm Chart pour Keycloak
## Ajout du repository Bitnami
Nous devons tout d'abord ajouter le repository Bitnami, le repository bitnami contient le helmchart keycloak necessaire au deploiment.

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

# Installation ou mise à jour du Helm Chart
Pour installer Keycloak, nous allons utiliser les valeurs créées précédemment, puis spécifier les valeurs. L'installation du Helm se fait alors via `bitnami/keycloak` avec la version `24.4.9` dans le namespace `keycloak` (création s'il n'y en a pas encore).

Dans le fichier `values.yaml`, ne pas oublier de mettre le `storageClassName` pour MicroK8s.

```sh
helm upgrade --install keycloak bitnami/keycloak --version 24.4.9 -n keycloak --create-namespace -f values.yaml
```

# Vérification de l'installation
Après que le Helm Chart soit appliqué, nous devons avoir un StatefulSet fonctionnel avec un Postgres opérationnel. 
Pour vérifier si Postgres et Keycloak sont fonctionnels, nous pouvons également faire un port-forward sur les instances et configurer Keycloak.

