# Installation de rabbitmq avec les fichier yaml
Nous avons besoins de plusieurs manifest pour installer rabbitmq

## Configmap
Contient la configuration de base pour notre cluster rabbit mq

## rabbit-rbac
Permet a notre utilisateur d'avoir un role qui permet de voir d'autre instance de rabbitmq

## Secret
Contient le ERLANG COOKIE, ce qui permet alors au instance de communiquer

## Statefulset
Contient la configuration de base de l'application, avec tout les ports definie.


# Lancement du cluster rabbitmq
Pour lancer le cluster , nous devons executer la commande suivante:
```sh
k create namespace rabbit

k apply -f configmap.yaml
k apply -f secret.yaml
k apply -f rabbit-rbac.yaml
k apply -f statefulset.yaml
```

