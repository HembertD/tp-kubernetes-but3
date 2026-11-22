# TP Kubernetes — Partie 2 : MySQL, WordPress et NGINX

> J'ai regroupé ici toute la partie 2 : la base **MySQL**, le service **WordPress** et le
> reverse proxy **NGINX**.
> Tout a été déployé et testé sur Minikube ; la chaîne complète fonctionne.
>
> 📸 Captures : [`img/partie2-mysql-variables.png`](img/partie2-mysql-variables.png)
> (variables MySQL), [`img/partie2-wordpress-variables.png`](img/partie2-wordpress-variables.png)
> (variables WordPress) et [`img/partie2-wordpress-installation.png`](img/partie2-wordpress-installation.png)
> (page d'installation via NGINX)

J'ai fait cette partie dans l'ordre MySQL → WordPress → NGINX, parce que chaque couche dépend de
la précédente : WordPress a besoin de MySQL, et NGINX a besoin du Service WordPress. Les
commandes et les captures ci-dessous correspondent à ce que j'ai réellement lancé sur mon
cluster Minikube.

---

## Sommaire

1. [Architecture globale](#1-architecture-globale)
2. [Partie A — MySQL](#2-partie-a--mysql)
3. [Partie B — WordPress](#3-partie-b--wordpress)
4. [Partie C — NGINX](#4-partie-c--nginx)
5. [Déploiement global et test de bout en bout](#5-déploiement-global-et-test-de-bout-en-bout)
6. [Réponses aux questions de vérification](#6-réponses-aux-questions-de-vérification)
7. [Erreurs fréquentes et diagnostic](#7-erreurs-fréquentes-et-diagnostic)
8. [Nettoyage](#8-nettoyage)
9. [Livrables](#9-livrables)
10. [Ce que j'ai retenu](#10-ce-que-jai-retenu)

---

## 1. Architecture globale

```text
Navigateur (PC)
   │  http://localhost:8080
   ▼
kubectl port-forward (8080 -> 8383)
   │
   ▼
namespace front
┌──────────────────────────────────────────────┐
│ Service nginx (ClusterIP)   8383 -> 80       │
│        │                                     │
│        ▼                                     │
│ Pods nginx x2 (reverse proxy)                │
│   default.conf monté depuis le ConfigMap     │
│   nginx-cm                                   │
└──────────────────────────────────────────────┘
   │  proxy_pass http://wordpress.middle.svc.cluster.local
   ▼
namespace middle
┌──────────────────────────────────────────────┐
│ Service wordpress (ClusterIP)   80 -> 80     │
│        │                                     │
│        ▼                                     │
│ Pod wordpress (image wordpress:latest)       │
│   /var/www/html monté sur le PVC             │
└──────────────────────────────────────────────┘
   │  WORDPRESS_DB_HOST=mysql.database.svc.cluster.local
   ▼
namespace database
┌──────────────────────────────────────────────┐
│ Service mysql (ClusterIP)   3306 -> 3306     │
│        │                                     │
│        ▼                                     │
│ Pod mysql-0 (StatefulSet)                    │
│   données dans /var/lib/mysql (PVC 1Gi)      │
└──────────────────────────────────────────────┘
```

Chaque namespace isole une couche :

| Namespace | Rôle | Ressources |
|---|---|---|
| `front` | Reverse proxy | Deployment/Service NGINX, ConfigMap `nginx-cm` |
| `middle` | Application | Deployment/Service WordPress, PVC, copies Secret/ConfigMap |
| `database` | Données | StatefulSet/Service MySQL, PVC, Secret/ConfigMap |

---

## 2. Partie A — MySQL

### 2.1 Vue d'ensemble

```text
                 namespace database
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   Service mysql (ClusterIP)                              │
│   DNS : mysql.database.svc.cluster.local:3306            │
│              │                                           │
│              ▼                                           │
│   StatefulSet mysql ──► Pod mysql-0                      │
│                          ├── image mysql:9.0             │
│                          ├── variables ConfigMap/Secret  │
│                          └── /var/lib/mysql              │
│                                   │                      │
│                                   ▼                      │
│                    PVC mysql-persistent-storage-mysql-0  │
│                    (1Gi, ReadWriteOnce, class standard)  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

Objectif : fournir une base MySQL persistante, accessible par le pod WordPress via le DNS
`mysql.database.svc.cluster.local`.

| Fichier | Ressource | Rôle |
|---|---|---|
| `TP/database/mysql-secret.yml` | `Secret` | Mots de passe (sensibles) |
| `TP/database/mysql-configmap.yml` | `ConfigMap` | Utilisateur et nom de base (non sensibles) |
| `TP/database/mysql-statefulset.yml` | `StatefulSet` | Le serveur MySQL + son stockage |
| `TP/database/mysql-service.yml` | `Service` | Point d'accès réseau stable sur le port 3306 |

### 2.2 `mysql-secret.yml` — les mots de passe

#### Génération des valeurs base64

```bash
echo -n 'wordpresspass' | base64   # -> d29yZHByZXNzcGFzcw==
echo -n 'rootpass'      | base64   # -> cm9vdHBhc3M=
```

> **Attention :** `-n` est important : sans lui, `echo` ajoute un retour à la ligne, qui serait encodé aussi
> et deviendrait une partie du mot de passe.

#### Fichier

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: database
type: Opaque
data:
  MYSQL_PASSWORD: d29yZHByZXNzcGFzcw==
  MYSQL_ROOT_PASSWORD: cm9vdHBhc3M=
```

#### Ce que ça fait, exactement

- `kind: Secret` crée un objet destiné aux données sensibles.
- `type: Opaque` = Secret « générique », sans format imposé.
- `namespace: database` place le Secret dans le namespace de MySQL. **Un pod ne peut lire que
  les Secrets/ConfigMaps de son propre namespace** : le namespace est donc indispensable.
- `data` attend des valeurs **encodées en base64**. Kubernetes les décode au moment de les
  injecter dans le conteneur.
- Les clés `MYSQL_PASSWORD` et `MYSQL_ROOT_PASSWORD` portent exactement les noms attendus par
  l'image officielle `mysql`.

#### Pourquoi un Secret plutôt qu'un ConfigMap ?

- Un Secret est destiné aux données sensibles : il peut être protégé par du RBAC plus fin,
  masqué dans certains affichages, et il n'est pas stocké en clair dans le YAML.
- **Attention :** **Base64 n'est pas du chiffrement** : n'importe qui peut décoder. Dans un vrai projet on
  utiliserait Sealed Secrets, SOPS, External Secrets, Vault, etc.
- On peut aussi écrire directement en clair avec `stringData:` (Kubernetes encode alors
  lui-même), mais le sujet demande explicitement `data` en base64.

#### Vérifier

```bash
kubectl get secret mysql-secret -n database
kubectl get secret mysql-secret -n database -o yaml
# Décoder une valeur
kubectl get secret mysql-secret -n database -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d
```

### 2.3 `mysql-configmap.yml` — les données non sensibles

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  namespace: database
data:
  MYSQL_USER: wordpress
  MYSQL_DATABASE: wordpress
```

#### Ce que ça fait

- `ConfigMap` stocke de la configuration **non confidentielle**.
- `MYSQL_USER: wordpress` : l'image MySQL créera l'utilisateur `wordpress`.
- `MYSQL_DATABASE: wordpress` : l'image créera la base `wordpress`.
- Comme pour le Secret, le ConfigMap est dans le namespace `database` pour être lisible par le
  pod MySQL.

#### Ce que fait l'image MySQL avec ces variables

Au **premier démarrage** (répertoire de données vide), l'entrypoint de l'image :

1. initialise la base système ;
2. crée la base indiquée par `MYSQL_DATABASE` ;
3. crée l'utilisateur indiqué par `MYSQL_USER` avec `MYSQL_PASSWORD` ;
4. donne à cet utilisateur tous les droits sur `MYSQL_DATABASE` ;
5. définit le mot de passe root avec `MYSQL_ROOT_PASSWORD` ;
6. redémarre définitivement le serveur.

> **Attention :** **Important** : ces variables ne sont prises en compte que lors de l'initialisation d'un
> volume vide. Si vous changez un mot de passe dans le Secret après coup, la base existante ne
> sera pas modifiée. Il faut alors changer le mot de passe avec `ALTER USER` ou supprimer le PVC
> pour repartir de zéro.

#### Vérifier

```bash
kubectl get configmap mysql-configmap -n database -o yaml
```

### 2.4 `mysql-statefulset.yml` — le serveur MySQL

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:9.0
          ports:
            - name: mysql
              containerPort: 3306
              protocol: TCP
          env:
            - name: MYSQL_USER
              valueFrom:
                configMapKeyRef:
                  name: mysql-configmap
                  key: MYSQL_USER
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-configmap
                  key: MYSQL_DATABASE
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-persistent-storage
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

#### Explication ligne par ligne

| Élément | Rôle |
|---|---|
| `apiVersion: apps/v1` | **Attention :** Un StatefulSet appartient au groupe `apps`, **pas** à `v1`. `apiVersion: v1` provoque une erreur. |
| `kind: StatefulSet` | Charge de travail avec identité stable (contrairement à un Deployment). |
| `metadata.name: mysql` | Nom du StatefulSet. |
| `metadata.namespace: database` | Isole la base dans son namespace. |
| `spec.serviceName: mysql` | Nom du Service associé, utilisé pour l'identité réseau stable des pods. |
| `replicas: 1` | Un seul serveur MySQL (le sujet impose 1). |
| `selector.matchLabels` | Le StatefulSet gère les pods portant `app: mysql`. |
| `template.metadata.labels` | Le pod créé porte bien `app: mysql`, ce qui le rend sélectionnable. |
| `containers[].name: mysql` | Nom du conteneur. |
| `image: mysql:9.0` | Image officielle MySQL 9.0. |
| `containerPort: 3306` | Port d'écoute de MySQL dans le conteneur (documentaire/réseau). |
| `env[].valueFrom.configMapKeyRef` | Injecte une variable depuis une clé précise du ConfigMap. |
| `env[].valueFrom.secretKeyRef` | Injecte une variable depuis une clé précise du Secret. |
| `volumeMounts.mountPath: /var/lib/mysql` | Répertoire où MySQL stocke ses données. |
| `volumeClaimTemplates` | Modèle de PVC : Kubernetes crée automatiquement un PVC par pod. |

#### Pourquoi un `StatefulSet` et pas un `Deployment` ?

Un Deployment est fait pour des applications **sans état** (nginx, WordPress…) : les pods sont
interchangeables et n'importe quel pod peut être supprimé/recréé.

MySQL est **avec état** :

- il a besoin d'un **stockage qui survive** au pod ;
- il a besoin d'une **identité stable** : le pod s'appelle toujours `mysql-0` (et non
  `mysql-5d68b47d97-xxxxx`) ;
- il a besoin d'un **démarrage ordonné** (un seul pod ici, mais le principe reste) ;
- `volumeClaimTemplates` associe **un PVC dédié à chaque pod** : `mysql-persistent-storage-mysql-0`.

Si le pod `mysql-0` est supprimé, le StatefulSet le recrée avec **le même nom** et **le même
PVC** : les données sont retrouvées.

#### Le stockage persistant

```yaml
volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

- `ReadWriteOnce` (RWO) : le volume ne peut être monté en lecture/écriture que par **un seul
  nœud** à la fois.
- `1Gi` : capacité demandée.
- Aucun `storageClassName` n'est précisé : Kubernetes utilise la **StorageClass par défaut**.
  Sur Minikube, c'est `standard` (provisioner `k8s.io/minikube-hostpath`), qui crée
  automatiquement le volume.
- Le PVC réellement créé s'appelle `mysql-persistent-storage-mysql-0` : c'est le nom du template
  + le nom du pod.

Vérification :

```bash
kubectl get pvc -n database
```

```text
NAME                               STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
mysql-persistent-storage-mysql-0   Bound    pvc-...    1Gi        RWO            standard
```

> **Astuce :** Le pod peut rester quelques secondes en `Pending` avec l'event
> `pod has unbound immediate PersistentVolumeClaims` : c'est normal, le temps que le
> provisioner crée le volume. Il passe ensuite en `Running`.

#### Le lien avec le Secret et le ConfigMap

- Si le Secret ou le ConfigMap n'existe pas, ou si une clé est mal orthographiée, le pod reste
  en `CreateContainerConfigError` (visible avec `kubectl describe pod`).
- On utilise `configMapKeyRef` / `secretKeyRef` (et non `envFrom`) pour **choisir précisément**
  les variables injectées.

### 2.5 `mysql-service.yml` — l'accès réseau

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: database
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - name: mysql
      protocol: TCP
      port: 3306
      targetPort: 3306
```

#### Ce que ça fait

- `ClusterIP` : le Service n'est accessible **qu'à l'intérieur du cluster** (pas depuis
  l'extérieur). C'est exactement ce qu'il faut pour une base de données.
- `selector: app: mysql` : le Service envoie le trafic vers les pods portant ce label.
- `port: 3306` : port exposé par le Service.
- `targetPort: 3306` : port réel du conteneur MySQL.
- Le Service fournit un **DNS stable** et une **IP virtuelle stable** (dans notre cas
  `10.101.201.207`), même si l'IP du pod change.

#### Nom DNS utilisé par WordPress

```text
mysql.database.svc.cluster.local
  │       │        │
  │       │        └── cluster.local = domaine du cluster
  │       └─────────── database = namespace
  └─────────────────── mysql = nom du Service
```

Test effectué :

```bash
kubectl run dns-test --rm -i --restart=Never --image=busybox:1.28 -n database -- \
  nslookup mysql.database.svc.cluster.local
```

Résultat :

```text
Name:      mysql.database.svc.cluster.local
Address 1: 10.101.201.207 mysql.database.svc.cluster.local
```

#### Vérifier / accéder

```bash
kubectl get svc -n database
kubectl get endpoints mysql -n database

# Redirection depuis le PC (utile pour tester avec un client MySQL local)
kubectl port-forward -n database svc/mysql 3306:3306
```

### 2.6 Déploiement et vérification MySQL

#### Commandes, dans l'ordre

```bash
# 1) Le namespace doit exister en premier
kubectl apply -f TP/database/namespace.yml

# 2) Les données de configuration avant le pod qui les consomme
kubectl apply -f TP/database/mysql-secret.yml
kubectl apply -f TP/database/mysql-configmap.yml

# 3) Le Service (peut être créé avant ou après le StatefulSet)
kubectl apply -f TP/database/mysql-service.yml

# 4) Le StatefulSet qui crée le pod et le PVC
kubectl apply -f TP/database/mysql-statefulset.yml

# 5) Attendre que MySQL soit prêt
kubectl rollout status statefulset/mysql -n database
```

> On peut aussi faire `kubectl apply -f TP/database/` pour appliquer tout le dossier d'un coup.
> L'ordre ci-dessus est surtout pédagogique : Secret/ConfigMap doivent exister avant que le pod
> ne démarre.

#### Résultat observé

```text
NAME                     READY   AGE
statefulset.apps/mysql   1/1     51s

NAME          READY   STATUS    RESTARTS   AGE
pod/mysql-0   1/1     Running   0          51s

NAME            TYPE        CLUSTER-IP       PORT(S)    AGE
service/mysql   ClusterIP   10.101.201.207   3306/TCP   51s

NAME                                                     STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/mysql-persistent-storage-mysql-0   Bound    pvc-...    1Gi        RWO            standard
```

#### 📸 Vérification des 4 variables d'environnement MySQL

```bash
kubectl exec -n database mysql-0 -- \
  printenv MYSQL_DATABASE MYSQL_USER MYSQL_PASSWORD MYSQL_ROOT_PASSWORD
```

Résultat :

```text
wordpress
wordpress
wordpresspass
rootpass
```

📸 Capture : [`img/partie2-mysql-variables.png`](img/partie2-mysql-variables.png)

| Variable | Source | Valeur |
|---|---|---|
| `MYSQL_DATABASE` | ConfigMap | `wordpress` |
| `MYSQL_USER` | ConfigMap | `wordpress` |
| `MYSQL_PASSWORD` | Secret | `wordpresspass` |
| `MYSQL_ROOT_PASSWORD` | Secret | `rootpass` |

Test supplémentaire de la base :

```bash
kubectl exec -n database mysql-0 -- \
  mysql -u wordpress -pwordpresspass -e "SHOW DATABASES; SELECT CURRENT_USER();"
```

```text
Database
information_schema
performance_schema
wordpress

CURRENT_USER()
wordpress@%
```

La base `wordpress` a bien été créée par l'entrypoint MySQL, et l'utilisateur `wordpress` peut
s'y connecter.

> **Astuce :** Au tout premier démarrage, une commande `mysql` peut échouer avec
> `Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock'` :
> le conteneur est lancé mais MySQL est encore en train de s'initialiser. Il suffit d'attendre
> le message `ready for connections` dans les logs :
> ```bash
> kubectl logs -n database mysql-0 -f
> ```
> Le statut `READY 1/1` signifie seulement que le conteneur tourne : sans `readinessProbe`,
> Kubernetes ne sait pas si MySQL est réellement prêt à répondre.

---

## 3. Partie B — WordPress

### 3.1 `wordpress-pv.yml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/kubernetes/wordpress
    type: DirectoryOrCreate
```

#### Ce que ça fait

| Élément | Explication |
|---|---|
| `kind: PersistentVolume` | Volume de stockage **au niveau du cluster** (pas de namespace). |
| `capacity.storage: 100Mi` | Taille annoncée du volume : 100 Mio. |
| `accessModes: ReadWriteMany` | Lecture/écriture par plusieurs nœuds (demandé par le sujet). |
| `storageClassName: manual` | Étiquette de classement. Le PVC devra demander la même classe pour se lier. |
| `persistentVolumeReclaimPolicy: Retain` | Si le PVC est supprimé, le volume et les données sont conservés. |
| `hostPath.path: /tmp/kubernetes/wordpress` | Stockage réel sur le nœud Minikube. `DirectoryOrCreate` crée le dossier s'il n'existe pas. |

> **Attention :** `hostPath` est un stockage **local au nœud**. Il est parfait pour un TP sur Minikube
> (mono-nœud), mais pas pour un vrai cluster multi-nœuds : le dossier ne serait pas partagé
> entre les nœuds. En production on utiliserait un vrai provisioner (NFS, Ceph, cloud…).
>
> **Attention :** Avec `hostPath`, le mode `ReadWriteMany` est **déclaré** mais pas réellement appliqué par
> Kubernetes : c'est le PVC qui a besoin de retrouver un PV avec le même mode.

#### Vérifier

```bash
kubectl get pv wordpress-pv
# STATUS: Available puis Bound après création du PVC
```

Le contenu réel se trouve **dans la VM Minikube** :

```bash
minikube ssh -- ls -la /tmp/kubernetes/wordpress
```

### 3.2 `wordpress-pvc.yml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: middle
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 100Mi
```

#### Ce que ça fait

- Un `PersistentVolumeClaim` est une **demande de stockage** faite par un namespace.
- Kubernetes (le control plane) cherche un PV qui satisfait **toutes** les exigences :
  - même `storageClassName` (`manual`) ;
  - capacité suffisante (`100Mi`) ;
  - mode d'accès compatible (`ReadWriteMany`).
- Si un PV correspond, il lie automatiquement le PVC au PV : `STATUS: Bound`.
- Le pod WordPress référence le **PVC**, jamais le PV directement. C'est une abstraction : le
  développeur demande « 100Mi en lecture/écriture », l'administrateur fournit le volume derrière.

#### Résultat observé

```text
NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS
wordpress-pvc   Bound    wordpress-pv   100Mi      RWX            manual
```

### 3.3 Copie du Secret et du ConfigMap MySQL dans `middle`

Fichiers ajoutés :

- `TP/middle/mysql-secret.yml` → Secret `mysql-secret` dans `middle`
- `TP/middle/mysql-configmap.yml` → ConfigMap `mysql-configmap` dans `middle`

Ils ont exactement le même contenu que ceux de `database`, seule la ligne
`namespace: middle` change.

#### Pourquoi cette duplication ?

Dans Kubernetes, **un pod ne peut référencer que les Secrets et ConfigMaps de son propre
namespace**. Le pod WordPress tourne dans `middle` : il ne peut donc pas lire directement le
Secret/ConfigMap qui sont dans `database`.

- Le **StatefulSet et le Service MySQL** restent dans `database` : WordPress les joint par le
  réseau (`mysql.database.svc.cluster.local`), pas par une référence d'objet.
- Le **Secret et le ConfigMap** doivent être dupliqués dans `middle` car ils sont injectés
  comme variables d'environnement dans le pod WordPress.

> C'est la réponse à la question « pourquoi les ressources MySQL ConfigMap sont à déployer au
> sein des namespaces middle et database ? ».

### 3.4 `wordpress-deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: middle
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: blog
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          env:
            - name: WORDPRESS_DB_USER
              valueFrom:
                configMapKeyRef:
                  name: mysql-configmap
                  key: MYSQL_USER
            - name: WORDPRESS_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: mysql-configmap
                  key: MYSQL_DATABASE
            - name: WORDPRESS_DB_HOST
              value: mysql.database.svc.cluster.local
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
            - name: WORDPRESS_CONFIG_EXTRA
              value: |
                define('WP_HOME', 'http://localhost:8080');
                define('WP_SITEURL', 'http://localhost:8080');
          volumeMounts:
            - name: wordpress-data
              mountPath: /var/www/html
      volumes:
        - name: wordpress-data
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

#### Explication détaillée

| Élément | Rôle |
|---|---|
| `replicas: 1` | Un seul pod WordPress. |
| `strategy: Recreate` | L'ancien pod est supprimé **avant** de créer le nouveau. Évite deux pods WordPress écrivant en même temps dans le même volume. |
| `selector.matchLabels: app: blog` | Le Deployment gère les pods `app: blog`. |
| `template.metadata.labels: app: blog` | Le pod porte le label sélectionné par le Deployment **et** par le Service. |
| `image: wordpress:latest` | Image officielle WordPress (Apache + PHP). |
| `containerPort: 80` | Apache écoute sur le port 80 dans le conteneur. |
| `WORDPRESS_DB_USER` | Lu depuis `MYSQL_USER` du ConfigMap → `wordpress`. |
| `WORDPRESS_DB_NAME` | Lu depuis `MYSQL_DATABASE` du ConfigMap → `wordpress`. |
| `WORDPRESS_DB_HOST` | DNS interne du Service MySQL. |
| `WORDPRESS_DB_PASSWORD` | Lu depuis `MYSQL_PASSWORD` du Secret → `wordpresspass`. |
| `WORDPRESS_CONFIG_EXTRA` | Code PHP ajouté à la configuration WordPress au démarrage. |
| `volumeMounts.mountPath: /var/www/html` | Le site complet (fichiers + uploads) est stocké sur le PVC. |
| `volumes.persistentVolumeClaim.claimName: wordpress-pvc` | Utilise le PVC créé précédemment. |

#### Pourquoi `WORDPRESS_CONFIG_EXTRA` est indispensable

Sans lui, WordPress génère ses URL à partir de ce qu'il reçoit. Derrière un `port-forward`,
WordPress peut produire des redirections vers `http://wordpress.middle.svc.cluster.local` ou
vers l'IP du pod, **inaccessibles depuis le navigateur du PC**.

On force donc :

```php
define('WP_HOME', 'http://localhost:8080');
define('WP_SITEURL', 'http://localhost:8080');
```

- `WP_HOME` = URL d'accueil du site.
- `WP_SITEURL` = URL de l'administration WordPress.
- `8080` doit correspondre au **port local du port-forward** utilisé pour NGINX. Si vous changez
  ce port, changez aussi ces deux lignes.

> **Astuce :** L'entrypoint officiel de l'image ne recopie pas littéralement ces lignes dans
> `wp-config.php` : il ajoute un `eval(getenv_docker('WORDPRESS_CONFIG_EXTRA', ''))`. Les
> constantes sont donc définies à l'exécution. C'est normal si un `grep WP_HOME wp-config.php`
> ne trouve rien.

#### Le montage du volume

L'image WordPress contient les fichiers du site dans `/usr/src/wordpress`. Au premier démarrage,
l'entrypoint détecte que `/var/www/html` est vide (car le PVC est vide) et **copie WordPress
dedans**. Ensuite, ces fichiers vivent sur le PVC : ils survivent à la suppression/recréation du
pod.

### 3.5 `wordpress-service.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: middle
spec:
  type: ClusterIP
  selector:
    app: blog
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

#### Ce que ça fait

- `ClusterIP` : accessible uniquement depuis le cluster. C'est NGINX qui l'expose vers
  l'extérieur via le `port-forward`.
- `selector: app: blog` : envoie le trafic aux pods WordPress.
- `port: 80` / `targetPort: 80` : le Service écoute sur 80 et redirige vers le port 80 du pod.
- DNS interne : `wordpress.middle.svc.cluster.local`.

#### Vérifier

```bash
kubectl get svc wordpress -n middle
kubectl get endpoints wordpress -n middle
```

`ENDPOINTS` doit contenir l'IP du pod WordPress (`10.244.0.7:80` dans notre cas). Si la liste est
vide, le selector ne correspond à aucun pod prêt.

### 3.6 Vérification des variables WordPress

```bash
POD=$(kubectl get pod -n middle -l app=blog -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n middle "$POD" -- \
  printenv WORDPRESS_DB_USER WORDPRESS_DB_NAME WORDPRESS_DB_HOST WORDPRESS_DB_PASSWORD
```

Résultat :

```text
wordpress
wordpress
mysql.database.svc.cluster.local
wordpresspass
```

📸 Capture : [`img/partie2-wordpress-variables.png`](img/partie2-wordpress-variables.png)

| Variable WordPress | Source | Valeur |
|---|---|---|
| `WORDPRESS_DB_USER` | ConfigMap `MYSQL_USER` | `wordpress` |
| `WORDPRESS_DB_NAME` | ConfigMap `MYSQL_DATABASE` | `wordpress` |
| `WORDPRESS_DB_HOST` | valeur fixe | `mysql.database.svc.cluster.local` |
| `WORDPRESS_DB_PASSWORD` | Secret `MYSQL_PASSWORD` | `wordpresspass` |

Vérifier que WordPress a bien copié ses fichiers sur le PVC :

```bash
kubectl exec -n middle "$POD" -- ls /var/www/html | head
minikube ssh -- ls -la /tmp/kubernetes/wordpress
```

---

## 4. Partie C — NGINX

### 4.1 `nginx-cm.yml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-cm
  namespace: front
data:
  default.conf: |
    server {
        listen       80;
        server_name  localhost;

        location / {
            proxy_pass         http://wordpress.middle.svc.cluster.local;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
        }

        access_log /var/log/nginx/access.log;
        error_log  /var/log/nginx/error.log;
    }
```

#### Ce que ça fait

| Directive | Rôle |
|---|---|
| `listen 80` | NGINX écoute sur le port 80 du conteneur. |
| `location /` | Toutes les URL sont concernées. |
| `proxy_pass http://wordpress.middle.svc.cluster.local` | Transfère la requête au Service WordPress (port 80 par défaut). |
| `proxy_set_header Host $host` | Conserve l'en-tête `Host` d'origine (`localhost:8080`). |
| `X-Real-IP` / `X-Forwarded-For` | Transmettent l'IP réelle du client. |
| `X-Forwarded-Proto` | Indique le protocole d'origine (http/https). |
| `proxy_http_version 1.1` | Active HTTP/1.1 entre NGINX et WordPress (keep-alive, performances). |

> **Attention :** Le fichier `TP/front/nginx-configmap.yml` présent au départ était un template incomplet
> (pas d'`apiVersion`, `kind`, `metadata`, `data`). Le fichier livrable de cette partie est
> **`nginx-cm.yml`**. L'ancien fichier n'est pas appliqué.

### 4.2 Montage dans le Deployment NGINX

Le Deployment `TP/front/nginx-deployment.yml` a été complété :

```yaml
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
              readOnly: true
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-cm
```

- Le ConfigMap est monté **comme un fichier** dans `/etc/nginx/conf.d/default.conf`.
- `subPath: default.conf` monte uniquement la clé `default.conf`, sans remplacer tout le dossier
  `/etc/nginx/conf.d` (et donc sans masquer d'autres fichiers de configuration).
- `readOnly: true` : NGINX n'a pas besoin d'écrire dans sa configuration.
- Pour que la modification soit prise en compte, il faut redémarrer les pods NGINX (rollout) :

```bash
kubectl apply -f TP/front/nginx-cm.yml
kubectl apply -f TP/front/nginx-deployment.yml
kubectl rollout status deployment/nginx -n front
```

### 4.3 Vérifier que la config est bien montée

```bash
kubectl exec -n front deploy/nginx -- cat /etc/nginx/conf.d/default.conf
kubectl exec -n front deploy/nginx -- nginx -t   # syntaxe OK
```

---

## 5. Déploiement global et test de bout en bout

### 5.1 Ordre de déploiement complet

```bash
# ---------- Namespaces ----------
kubectl apply -f TP/front/namespace.yml
kubectl apply -f TP/middle/namespace.yml
kubectl apply -f TP/database/namespace.yml

# ---------- MySQL (database) ----------
kubectl apply -f TP/database/mysql-secret.yml
kubectl apply -f TP/database/mysql-configmap.yml
kubectl apply -f TP/database/mysql-service.yml
kubectl apply -f TP/database/mysql-statefulset.yml
kubectl rollout status statefulset/mysql -n database

# ---------- WordPress (middle) ----------
kubectl apply -f TP/middle/wordpress-pv.yml
kubectl apply -f TP/middle/wordpress-pvc.yml
kubectl apply -f TP/middle/mysql-secret.yml
kubectl apply -f TP/middle/mysql-configmap.yml
kubectl apply -f TP/middle/wordpress-service.yml
kubectl apply -f TP/middle/wordpress-deployment.yml
kubectl rollout status deployment/wordpress -n middle

# ---------- NGINX (front) ----------
kubectl apply -f TP/front/nginx-cm.yml
kubectl apply -f TP/front/nginx-deployment.yml
kubectl rollout status deployment/nginx -n front
```

> On peut aussi appliquer dossier par dossier : `kubectl apply -f TP/database/`,
> `kubectl apply -f TP/middle/`, `kubectl apply -f TP/front/`. L'ordre ci-dessus est
> pédagogique : les dépendances (namespace, Secret/ConfigMap, PVC) doivent exister avant les
> pods qui les consomment.

### 5.2 Résultat observé

```text
namespace middle :
pod/wordpress-...   1/1   Running
service/wordpress   ClusterIP   10.100.251.58   80/TCP
persistentvolumeclaim/wordpress-pvc   Bound   wordpress-pv   100Mi   RWX   manual

namespace front :
pod/nginx-... (x2)  1/1   Running
service/nginx       ClusterIP   10.104.132.191   8383/TCP
configmap/nginx-cm  1

namespace database :
pod/mysql-0                1/1   Running
statefulset.apps/mysql     1/1
service/mysql              ClusterIP   10.101.201.207   3306/TCP
persistentvolumeclaim/mysql-persistent-storage-mysql-0   Bound   1Gi   RWO   standard
```

### 5.3 Accès au site

#### a) Tester WordPress directement

```bash
kubectl port-forward -n middle svc/wordpress 8081:80
```

Puis <http://localhost:8081> : WordPress redirige vers l'installation.

> On utilise `8081` pour ne pas entrer en conflit avec le port `8080` réservé à NGINX.

#### b) Tester la chaîne complète via NGINX (le vrai test du reverse proxy)

```bash
kubectl port-forward -n front svc/nginx 8080:8383
```

Puis <http://localhost:8080>.

```bash
curl -I http://localhost:8080/
```

```text
HTTP/1.1 302 Found
Server: nginx/1.27.1
Location: http://localhost:8080/wp-admin/install.php
X-Powered-By: PHP/8.3.36
```

```bash
curl -L http://localhost:8080/
```

```text
HTTP final: 200 | url: http://localhost:8080/wp-admin/install.php
<title>WordPress &rsaquo; Installation</title>
```

La requête a donc bien suivi le chemin :

```text
Navigateur -> NGINX (8080->8383) -> Service WordPress (80) -> Pod WordPress -> Service MySQL -> MySQL
```

📸 Capture de la page d'installation : [`img/partie2-wordpress-installation.png`](img/partie2-wordpress-installation.png)

#### Pourquoi deux port-forward différents ?

- `8080 -> 8383` : on entre par **NGINX**, qui applique sa configuration de reverse proxy. C'est
  le test complet.
- `8081 -> 80` : on entre directement dans **WordPress**, en contournant NGINX. Utile pour
  isoler un problème (si WordPress répond en direct mais pas via NGINX, le problème vient de
  NGINX).

---

## 6. Réponses aux questions de vérification

### ❓ Les différentes ressources sont-elles déployées dans le bon namespace ? Quelle commande ?

```bash
kubectl get all,pvc,secret,configmap -n database
kubectl get all,pvc,secret,configmap -n middle
kubectl get all,configmap -n front

# Vue globale
kubectl get pods -A
kubectl get all -A
```

Les PV sont **cluster-scoped** : `kubectl get pv` (sans `-n`).

Attendu :

- `front` : Deployment/Service/Pods NGINX + ConfigMap `nginx-cm`
- `middle` : Deployment/Service/Pod WordPress + PVC + Secret/ConfigMap MySQL
- `database` : StatefulSet/Service/Pod MySQL + PVC + Secret/ConfigMap MySQL
- cluster : PV `wordpress-pv`

### ❓ Pourquoi les ressources MySQL ConfigMap et MySQL sont-elles à déployer dans les namespaces `middle` et `database` ?

- Les ressources **MySQL elles-mêmes** (StatefulSet, Service, volume) vont dans `database` :
  c'est l'isolation prévue par le TP (front / middle / database).
- Le **Secret et le ConfigMap MySQL sont dupliqués dans `middle`** car **Kubernetes ne permet
  pas à un pod de référencer un Secret ou un ConfigMap situé dans un autre namespace**. Le pod
  WordPress tourne dans `middle` ; pour lire `MYSQL_USER`, `MYSQL_DATABASE` et
  `MYSQL_PASSWORD`, il lui faut une copie de ces objets dans `middle`.
- En revanche, WordPress n'a **pas** besoin d'une copie du StatefulSet ni du Service MySQL : il
  joint la base par le réseau, via `mysql.database.svc.cluster.local`.

### ❓ Les pods sont-ils correctement démarrés ? Quelle commande ?

```bash
kubectl get pods -n database
kubectl get pods -n middle
kubectl get pods -n front
kubectl get pods -A

kubectl rollout status statefulset/mysql -n database
kubectl rollout status deployment/wordpress -n middle
kubectl rollout status deployment/nginx -n front
```

Un pod correct affiche `READY 1/1` et `STATUS Running`. Vérifier aussi que les Services ont des
endpoints :

```bash
kubectl get endpoints -n database
kubectl get endpoints -n middle
kubectl get endpoints -n front
```

### ❓ Lorsqu'un pod est en erreur, comment accéder aux events du namespace et aux logs du pod ?

```bash
# Events du namespace (du plus ancien au plus récent)
kubectl get events -n <namespace> --sort-by=.lastTimestamp

# Détail complet du pod : events, état des conteneurs, volumes, image, etc.
kubectl describe pod <pod> -n <namespace>

# Logs du conteneur
kubectl logs <pod> -n <namespace>

# Logs du conteneur précédent (après un crash/redémarrage)
kubectl logs <pod> -n <namespace> --previous

# Suivre les logs en direct
kubectl logs <pod> -n <namespace> -f
```

Pour NGINX, vérifier aussi la config et les logs d'erreur :

```bash
kubectl exec -n front deploy/nginx -- nginx -t
kubectl logs -n front deploy/nginx
```

En cas de problème, l'ordre d'investigation typique est :
`get pods` → `describe pod` → `get events` → `logs` → éventuellement `exec` dans le pod.

---

## 7. Erreurs fréquentes et diagnostic

| Symptôme | Cause probable | Vérification / correction |
|---|---|---|
| `no matches for kind "StatefulSet" in version "v1"` | `apiVersion: v1` au lieu de `apps/v1` | Mettre `apiVersion: apps/v1` |
| `CreateContainerConfigError` | Secret/ConfigMap absent ou clé mal orthographiée | `kubectl describe pod` ; vérifier le namespace et les noms de clés |
| `illegal base64 data at input byte ...` | Valeur du Secret non encodée | Utiliser `echo -n '...' \| base64` |
| PVC `Pending` | Aucun PV compatible (classe, taille, mode) | `kubectl get pv,pvc` ; vérifier `storageClassName` et `accessModes` |
| `Access denied for user` | Mot de passe changé après l'initialisation | Le Secret n'est lu qu'au premier démarrage : modifier l'utilisateur SQL ou recréer le PVC |
| Le pod MySQL est `Running` mais ne répond pas | MySQL encore en cours d'initialisation | Attendre `ready for connections` dans les logs |
| `Error establishing a database connection` | Mauvais host/user/password, MySQL pas prêt | `kubectl exec ... printenv`, `kubectl get endpoints mysql -n database`, logs MySQL |
| NGINX `502 Bad Gateway` | Service WordPress sans endpoint, DNS faux, pod non prêt | `kubectl get endpoints wordpress -n middle` ; `kubectl logs -n front deploy/nginx` |
| NGINX `404` | `default.conf` non monté ou mauvais chemin | `kubectl exec -n front deploy/nginx -- cat /etc/nginx/conf.d/default.conf` |
| Redirection vers une URL inaccessible | `WP_HOME`/`WP_SITEURL` absents ou mauvais port | Vérifier `WORDPRESS_CONFIG_EXTRA` et le port du port-forward |
| Pod WordPress bloqué en `ContainerCreating` | Volume pas encore attaché / image en cours de pull | `kubectl describe pod` |
| Les données WordPress disparaissent | PVC non monté ou mauvais `mountPath` | Vérifier `volumeMounts` et `claimName` |

### Ordre de diagnostic

```text
kubectl get pods -A
   → kubectl describe pod <pod> -n <ns>
      → kubectl get events -n <ns>
         → kubectl logs <pod> -n <ns>
            → kubectl exec -it <pod> -n <ns> -- sh
```

---

## 8. Nettoyage

```bash
# ---------- MySQL ----------
kubectl delete -f TP/database/mysql-statefulset.yml
kubectl delete -f TP/database/mysql-service.yml
kubectl delete -f TP/database/mysql-configmap.yml
kubectl delete -f TP/database/mysql-secret.yml

# ---------- WordPress ----------
kubectl delete -f TP/middle/wordpress-deployment.yml
kubectl delete -f TP/middle/wordpress-service.yml
kubectl delete -f TP/middle/mysql-configmap.yml
kubectl delete -f TP/middle/mysql-secret.yml
kubectl delete -f TP/middle/wordpress-pvc.yml
kubectl delete -f TP/middle/wordpress-pv.yml

# ---------- NGINX ----------
kubectl delete -f TP/front/nginx-deployment.yml
kubectl delete -f TP/front/nginx-cm.yml

# Attention : supprimer un namespace supprime TOUT ce qu'il contient (dont les PVC)
kubectl delete -f TP/database/namespace.yml
kubectl delete -f TP/middle/namespace.yml
kubectl delete -f TP/front/namespace.yml
```

> **Attention :** Supprimer un StatefulSet **ne supprime pas** automatiquement ses PVC (politique de
> rétention par défaut). Pour effacer réellement les données MySQL :
> ```bash
> kubectl delete pvc mysql-persistent-storage-mysql-0 -n database
> ```
>
> **Attention :** Le PV `wordpress-pv` a une politique `Retain` : il n'est pas supprimé avec le PVC. Il faut
> le supprimer explicitement (`kubectl delete pv wordpress-pv`). Le dossier
> `/tmp/kubernetes/wordpress` reste aussi sur le nœud Minikube.

---

## 9. Livrables

### MySQL

- [x] `TP/database/mysql-secret.yml`
- [x] `TP/database/mysql-configmap.yml`
- [x] `TP/database/mysql-statefulset.yml`
- [x] `TP/database/mysql-service.yml`
- [x] Vérification des 4 variables d'environnement MySQL (section 2.6)
- [x] Capture : `img/partie2-mysql-variables.png`

### WordPress

- [x] `TP/middle/wordpress-pv.yml`
- [x] `TP/middle/wordpress-pvc.yml`
- [x] `TP/middle/wordpress-deployment.yml`
- [x] `TP/middle/wordpress-service.yml`
- [x] `TP/middle/mysql-secret.yml` et `TP/middle/mysql-configmap.yml` (copies pour WordPress)
- [x] Vérification des 4 variables WordPress (section 3.6)
- [x] Capture : `img/partie2-wordpress-variables.png`

### NGINX

- [x] `TP/front/nginx-cm.yml`
- [x] `TP/front/nginx-deployment.yml` mis à jour (montage du ConfigMap)
- [x] Capture de la page d'installation : [`img/partie2-wordpress-installation.png`](img/partie2-wordpress-installation.png)

### Document

- [x] Liste des commandes utilisées (ce document)
- [x] Réponses aux questions de vérification (section 6)
- [x] Chaîne complète testée : NGINX → WordPress → MySQL

---

## 10. Ce que j'ai retenu

C'est la partie qui m'a pris le plus de temps, surtout à cause de l'initialisation de MySQL.
Au début, `kubectl exec ... mysql` échouait alors que le pod était `Running` : en fait MySQL
n'était pas encore prêt, et le `READY 1/1` ne veut pas dire que le serveur accepte déjà les
connexions. Depuis, je regarde les logs (`kubectl logs -f`) avant de lancer une commande SQL.

Le deuxième point, c'est le namespace : le Secret et le ConfigMap MySQL doivent être copiés dans
`middle`, sinon WordPress ne les voit pas. C'est logique une fois qu'on a compris que les
Secrets/ConfigMaps sont eux aussi limités à un namespace, mais sur le moment j'ai cherché
pourquoi le pod WordPress ne démarrait pas.

Pour la capture des variables, `printenv` est plus lisible qu'un `env` complet : on voit
directement les quatre valeurs demandées, sans le bruit des variables système.
