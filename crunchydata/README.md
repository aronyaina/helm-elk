Voici la documentation corrigée et mise en forme pour une meilleure lisibilité et un format copiable.

# Installation de l'opérateur CrunchyDB
## Clonage du répertoire Crunchy Postgres Examples

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

## Installation de l'opérateur Crunchy Data
Installer l'opérateur avec Kustomize Kubernetes.
```sh
kubectl apply -k new_dir/install --server-side --force-conflicts
```

Après l'installation, nous avons alors l'opérateur Crunchy Data fonctionnel.

# Installation de Postgres
Pour installer Postgres, nous devons nous rendre dans le dossier `postgres`, et modifier le fichier suivant la configuration souhaitée. Étant donné que nous sommes sous MicroK8s, nous devons prendre en compte l'ajout de `storageClassName: microk8s-hostpath` comme ci-dessous :

Ajouter le code dans le backup et le volume Postgres pour que le statut ne reste pas sur `pending`.

```sh
cd new_dir/postgres 
nvim postgres.yaml
```

Puis, ajouter la ligne du `storageClassName` pour que le PVC assigne automatiquement le PV requis, et aussi pour que les pods ne restent pas en mode `pending`.

```yaml
storageClassName: microk8s-hostpath
```

Puis lancer le cluster Postgres avec la commande :

```sh
kubectl apply -k new_dir/postgres
```

# Installation de PgAdmin
Pour installer PgAdmin, nous devons également créer un `microk8s-hostpath` et ensuite installer PgAdmin.

```yaml
storageClassName: microk8s-hostpath
```

Pour confirmer que notre base de données est fonctionnelle, nous allons installer PgAdmin avec la commande :
```sh
kubectl apply -k new_dir/pgadmin
```


# Vérification des mots de passe et utilisateurs
Pour vérifier les mots de passe et utilisateurs, nous avons des secrets créés automatiquement contenant le mot de passe et l'utilisateur de chaque application.


# Enfin pour pouvoir entrer et verifier si tout fonctionne bien
Nous allons utiliser la fonctionnalite port-forwarding pour verifier les differentes instances
