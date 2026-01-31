# 🚀 Mini-Projet Kubernetes — PayMyBuddy

Déploiement de l'application **PayMyBuddy** (SpringBoot + MySQL) en utilisant des manifests Kubernetes sans Helm.
Le but est de comprendre en détail comment déployer une application dans un cluster Kubernetes en écrivant
chaque manifest à la main.

---

## 📋 Prérequis

- Un cluster Kubernetes fonctionnel
- `kubectl` configuré et connecté au cluster
- Docker installé sur le nœud pour le build de l'image
- Accès SSH au nœud pour le volume `hostPath`

---

## 🏗️ Structure du projet

```
.
├── ns-paymybuddy.yml              # Définition du namespace
├── mysql-pv.yml                   # PersistentVolume (stockage sur le nœud via hostPath)
├── mysql-pvc.yml                  # PersistentVolumeClaim (demande de stockage)
├── paymybuddy-secrets.yml         # Secret Kubernetes (mot de passe MySQL en base64)
├── paymybuddy-configmap.yml       # ConfigMap (script SQL d'initialisation de la BDD)
├── mysql-deployment.yml           # Deployment du conteneur MySQL
├── mysql-clusterip.yml            # Service ClusterIP pour exposer MySQL en interne
├── paymybuddy-deployment.yaml     # Deployment de l'application SpringBoot
├── paymybuddy-nodeport.yml        # Service NodePort pour exposer PayMyBuddy vers l'extérieur
└── PayMyBuddy/                    # Dossier source de l'application SpringBoot
```

---

## ⚡ Déploiement étape par étape

### 1. Build de l'image Docker

Avant de déployer sur Kubernetes, il faut d'abord construire l'image Docker de l'application SpringBoot.
On se place dans le dossier `PayMyBuddy` qui contient le `Dockerfile` et on lance le build.

```bash
$ cd PayMyBuddy
$ docker build -t paymybuddy:latest .
[+] Building 12.9s (7/8)
 => [1/3] FROM docker.io/library/amazoncorretto:17-alpine       9.1s
 => [2/3] WORKDIR /app                                           0.4s
 => [3/3] COPY target/paymybuddy.jar paymybuddy.jar              1.1s
 => => naming to docker.io/library/paymybuddy:latest             0.0s
```

On vérifie que l'image a bien été créée :

```bash
$ docker images | grep paymybuddy
paymybuddy   latest   8f8e9fa9b48e   14 seconds ago   340MB
```

On retourne au répertoire racine du projet :

```bash
$ cd ..
$ ls -rtlh
total 40K
-rw-r--r--. 1 root root  163 Jan 31 20:30 paymybuddy-secrets.yml
-rw-r--r--. 1 root root  237 Jan 31 20:30 paymybuddy-nodeport.yml
-rw-r--r--. 1 root root  877 Jan 31 20:30 paymybuddy-deployment.yaml
-rw-r--r--. 1 root root 3.7K Jan 31 20:30 paymybuddy-configmap.yml
-rw-r--r--. 1 root root  111 Jan 31 20:30 ns-paymybuddy.yml
-rw-r--r--. 1 root root  245 Jan 31 20:30 mysql-pvc.yml
-rw-r--r--. 1 root root  302 Jan 31 20:30 mysql-pv.yml
-rw-r--r--. 1 root root 1.1K Jan 31 20:30 mysql-deployment.yml
-rw-r--r--. 1 root root  209 Jan 31 20:30 mysql-clusterip.yml
-rw-r--r--. 1 root root 1.4K Jan 31 20:30 README.md
drwxr-xr-x. 5 root root   78 Jan 31 20:34 PayMyBuddy
```

---

### 2. Création du Namespace

Un namespace permet d'isoler les ressources Kubernetes. Toutes nos ressources seront déployées dans le namespace `paymybuddy`.

```bash
$ kubectl apply -f ns-paymybuddy.yml
namespace/paymybuddy created

$ kubectl get ns paymybuddy
NAME         STATUS   AGE
paymybuddy   Active   25s
```

Le namespace est bien créé et actif.

---

### 3. Création du PersistentVolume (PV)

Le PersistentVolume représente un espace de stockage physique sur le nœud. On utilise un `hostPath` qui stocke les données MySQL directement dans le répertoire `/data/mysql` du nœud.
Cela permet de persister les données même si le pod MySQL est redémarré.

```bash
$ kubectl apply -f mysql-pv.yml
persistentvolume/mysql-pv created

$ kubectl get pv mysql-pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
mysql-pv   1Gi        RWO            Delete           Available           manual         10s
```

Le PV est en statut `Available`, il attend d'être utilisé.

---

### 4. Création du PersistentVolumeClaim (PVC)

Le PVC est une demande de stockage émise par un pod. Il se lie automatiquement à un PV compatible.
Dans notre cas, il se lie au PV `mysql-pv` créé précédemment.

```bash
$ kubectl apply -f mysql-pvc.yml
persistentvolumeclaim/mysql-pvc created

$ kubectl get pvc mysql-pvc -n paymybuddy
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Bound    mysql-pv   1Gi        RWO            manual         2s

$ kubectl get pv mysql-pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   AGE
mysql-pv   1Gi        RWO            Delete           Bound    paymybuddy/mysql-pvc   manual         32s
```

Le PVC est bien en statut `Bound` (lié) au PV `mysql-pv`. Le stockage est prêt à être utilisé.

---

### 5. Création du ConfigMap

Le ConfigMap contient le script SQL `init.sql` qui sera utilisé pour initialiser la base de données MySQL
au premier démarrage. Ce script crée les tables (`user`, `bank_account`, `connection`, `transaction`)
et insère les données initiales.

```bash
$ kubectl apply -f paymybuddy-configmap.yml
configmap/mysql-init-script created

$ kubectl get configmap -n paymybuddy
NAME                DATA   AGE
mysql-init-script   1      1s
```

---

### 6. Création du Secret

Le Secret contient le mot de passe MySQL encodé en base64. On évite de mettre le mot de passe en clair dans les manifests pour raisons de sécurité.

```bash
$ kubectl apply -f paymybuddy-secrets.yml
secret/paymybuddy-secrets created

$ kubectl get secrets -n paymybuddy
NAME                  TYPE                                  DATA   AGE
default-token-97mwf   kubernetes.io/service-account-token   3      4m8s
paymybuddy-secrets    Opaque                                1      9s

$ kubectl describe secrets paymybuddy-secrets -n paymybuddy
Name:         paymybuddy-secrets
Namespace:    paymybuddy
Type:         Opaque
Data
====
mysql_root_password:  8 bytes
```

On peut voir que le secret contient bien la clé `mysql_root_password` (8 bytes correspond à notre mot de passe encodé).

---

### 7. Déploiement MySQL

On applique le Deployment MySQL qui :
- Utilise l'image `mysql:8.0`
- Monte le PVC pour persister les données dans `/var/lib/mysql`
- Monte le ConfigMap dans `/docker-entrypoint-initdb.d/` pour exécuter le script d'init au premier démarrage
- Récupère le mot de passe depuis le Secret

```bash
$ kubectl apply -f mysql-deployment.yml
deployment.apps/mysql created

$ kubectl get deploy -n paymybuddy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mysql   0/1     1            0           9s

$ kubectl get pods -n paymybuddy
NAME                     READY   STATUS    RESTARTS   AGE
mysql-5788b48887-9jf6t   1/1     Running   0          25s
```

Le pod MySQL est bien en statut `Running`.

#### Vérification des logs MySQL

On vérifie dans les logs que le script d'initialisation a bien été exécuté :

```bash
$ kubectl logs mysql-5788b48887-9jf6t -n paymybuddy
...
2026-01-31 21:05:22+00:00 [Note] [Entrypoint]: Initializing database files
...
2026-01-31 21:05:37+00:00 [Note] [Entrypoint]: Database files initialized
2026-01-31 21:05:37+00:00 [Note] [Entrypoint]: Starting temporary server
...
2026-01-31 21:05:40+00:00 [Note] [Entrypoint]: Creating database db_paymybuddy
2026-01-31 21:05:41+00:00 [Note] [Entrypoint]: /usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/init.sql
...
2026-01-31 21:05:43+00:00 [Note] [Entrypoint]: MySQL init process done. Ready for start up.
...
2026-01-31T21:05:44.071271Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. port: 3306
```

La ligne clé est : `running /docker-entrypoint-initdb.d/init.sql` qui confirme que le script a été exécuté avec succès.

> **Note importante** : Le script `init.sql` ne s'exécute qu'une seule fois, à la première initialisation de MySQL, lorsque le répertoire `/var/lib/mysql` est encore vide. Si vous devez relancer l'initialisation, vous devez supprimer le PV et le PVC puis les recréer.

#### Vérification de la base de données et des tables

On se connecte directement au conteneur MySQL pour vérifier que la base et les tables ont bien été créées :

```bash
$ kubectl exec -it deployment/mysql -n paymybuddy -- bash -c 'mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SHOW DATABASES;"'
+--------------------+
| Database           |
+--------------------+
| db_paymybuddy      |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

$ kubectl exec -it deployment/mysql -n paymybuddy -- bash -c 'mysql -uroot -p${MYSQL_ROOT_PASSWORD} db_paymybuddy -e "SHOW TABLES;"'
+-------------------------+
| Tables_in_db_paymybuddy |
+-------------------------+
| bank_account            |
| connection              |
| transaction             |
| user                    |
+-------------------------+
```

La base `db_paymybuddy` existe et les 4 tables sont bien créées.

---

### 8. Service ClusterIP pour MySQL

Le service ClusterIP permet aux autres pods du cluster de se connecter à MySQL via une adresse IP interne stable.
L'application PayMyBuddy utilisera cette IP pour se connecter à la base de données.

```bash
$ kubectl apply -f mysql-clusterip.yml
service/mysql created

$ kubectl get svc -n paymybuddy
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
mysql   ClusterIP   10.109.213.107   <none>        3306/TCP   17s
```

MySQL est maintenant accessible à l'adresse `10.109.213.107:3306` à l'intérieur du cluster.

---

### 9. Déploiement PayMyBuddy

On déploie l'application SpringBoot qui se connecte à MySQL via le service ClusterIP créé précédemment.
Le Deployment utilise l'image `paymybuddy:latest` buildée au début.

```bash
$ kubectl apply -f paymybuddy-deployment.yaml
deployment.apps/paymybuddy created

$ kubectl get deploy -n paymybuddy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
mysql        1/1     1            1           2m21s
paymybuddy   1/1     1            1           4s

$ kubectl get pods -n paymybuddy
NAME                          READY   STATUS    RESTARTS   AGE
mysql-5788b48887-9jf6t        1/1     Running   0          2m43s
paymybuddy-58f8d76689-9rcfl   1/1     Running   0          26s
```

Les deux pods sont en statut `Running` et `READY 1/1`.

---

### 10. Service NodePort pour PayMyBuddy

Le service NodePort expose l'application vers l'extérieur du cluster. Il attribue un port sur chaque nœud
du cluster qui redirige le trafic vers le pod PayMyBuddy.

```bash
$ kubectl apply -f paymybuddy-nodeport.yml
service/paymybuddy created

$ kubectl get svc -n paymybuddy
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mysql        ClusterIP   10.109.213.107   <none>        3306/TCP         57s
paymybuddy   NodePort    10.102.82.218    <none>        8080:30088/TCP   2s
```

Le service expose le port `8080` de l'application sur le port `30088` du nœud.

---

## 🌐 Accès à l'application

L'application est accessible depuis un navigateur via :
```bash
http://<IP-du-nœud>:30088
```

Voici quelques captures d'écran de l'application en cours d'exécution :

![Capture 1 - Ecran de connexion](asset/test_appli1.JPG)

![Capture 2 - Inscription OK](asset/test_appli2.JPG)

![Capture 3 - Authentification OK](asset/test_appli3.JPG)

![Capture 4 - Ecran de transaction](asset/test_appli4.JPG)

---

## 🏅 Résumé final

À la fin du déploiement, voici l'état de toutes les ressources :

| Ressource | Statut | Description |
|-----------|--------|-------------|
| Namespace `paymybuddy` | ✅ Active | Espace isolé pour nos ressources |
| PV `mysql-pv` | ✅ Bound | Stockage physique sur le nœud (`/data/mysql`) |
| PVC `mysql-pvc` | ✅ Bound | Lié au PV `mysql-pv` |
| ConfigMap `mysql-init-script` | ✅ Créé | Script SQL d'initialisation |
| Secret `paymybuddy-secrets` | ✅ Créé | Mot de passe MySQL (base64) |
| Deployment `mysql` | ✅ 1/1 Ready | Base de données MySQL |
| Service `mysql` (ClusterIP) | ✅ 3306/TCP | Accès interne à MySQL |
| Deployment `paymybuddy` | ✅ 1/1 Ready | Application SpringBoot |
| Service `paymybuddy` (NodePort) | ✅ 8080:30088/TCP | Accès externe à l'application |
