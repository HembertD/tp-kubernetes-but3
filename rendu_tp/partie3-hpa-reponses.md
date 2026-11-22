# TP Kubernetes — Partie 3 : HPA (étapes 1 à 9)

> J'ai traité toute la section HPA : création du Deployment, requests/limits, création du HPA,
> exposition du Service, vérification, test de charge et ajustements.
>
> Fichiers : [`TP/hpa/application.yml`](../TP/hpa/application.yml) et
> [`TP/hpa/hpa-application.yml`](../TP/hpa/hpa-application.yml)

J'ai d'abord vérifié que `metrics-server` répondait, puis j'ai appliqué le Deployment et le HPA.
Mon premier essai avec une cible à 50 % ne déclenchait rien : la charge restait pile à 50m, donc
l'HPA considérait qu'on était à la cible. Je suis repassé à 30 % et j'ai refait la charge ; c'est
ce passage que je détaille ci-dessous.

---

## 0. Pré-requis : metrics-server

Un HPA a besoin de connaître la consommation réelle des pods. Sans serveur de métriques, la
colonne `TARGETS` reste à `<unknown>/30%`.

```bash
minikube addons enable metrics-server
kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
```

Vérification :

```text
NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
minikube   116m         1%       1400Mi          9%
```

> **Astuce :** `metrics-server` collecte les métriques CPU/mémoire des pods et des nœuds. C'est lui qui
> alimente l'HPA.

---

## 1. `application.yml` — le Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-application
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mon-application
  template:
    metadata:
      labels:
        app: mon-application
    spec:
      containers:
        - name: mon-application
          image: nginx:1.27.1-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

### Explication

| Élément | Rôle |
|---|---|
| `kind: Deployment` | Gère un ou plusieurs pods interchangeables. |
| `metadata.name: mon-application` | Nom demandé par le sujet ; c'est aussi la cible du HPA. |
| `replicas: 1` | On démarre avec un seul pod ; l'HPA fera varier ce nombre. |
| `selector.matchLabels` | Le Deployment gère les pods portant `app: mon-application`. |
| `template.metadata.labels` | Le pod porte le label sélectionné. |
| `image: nginx:1.27.1-alpine` | Image légère, déjà utilisée dans la partie 1. |
| `containerPort: 80` | NGINX écoute sur le port 80. |
| `resources.requests` | **Ressources garanties** au pod ; sert de base au calcul des % de l'HPA. |
| `resources.limits` | **Plafond** que le pod ne peut pas dépasser. |

> **Point clé :** pour un HPA basé sur le CPU, la `requests.cpu` est **obligatoire**. Sans
> elle, Kubernetes ne sait pas calculer un pourcentage d'utilisation.

---

## 2. Requests et limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### Signification des valeurs

- `100m` = **0,1 CPU** (100 millicores). C'est la garantie minimale réservée au pod.
- `200m` = **0,2 CPU** : le pod ne pourra pas consommer plus de 20 % d'un cœur.
- `128Mi` / `256Mi` : mémoire garantie / mémoire maximale.

### Pourquoi ces valeurs ?

L'image officielle `nginx` ne publie pas de « minimum recommandé » officiel. J'ai donc choisi un
profil **volontairement bas et raisonnable**, adapté à un test local :

- `requests.cpu: 100m` : NGINX au repos consomme très peu (quelques millicores). 100m laisse de
  la marge tout en restant assez bas pour que la charge de test fasse dépasser le seuil.
- `limits.cpu: 200m` : deux fois la request, ce qui évite qu'un pod monopolise le CPU du nœud.
- `requests.memory: 128Mi` / `limits.memory: 256Mi` : suffisant pour NGINX et cohérent avec la
  taille de l'image.

> **Attention :** Ces valeurs sont un choix pédagogique. En production, on les dimensionne à partir de
> mesures réelles (`kubectl top`, profiling, tests de charge).

### Cohérence avec l'HPA

J'ai d'abord testé une cible à **50 %** de la request CPU. Avec une boucle `wget` unique, la
consommation restait autour de **50m**, soit exactement 50 % : l'HPA ne déclenchait rien car il
faut **dépasser** la cible, pas l'atteindre.

J'ai donc ajusté la cible à **30 %** :

```text
seuil = 30 % × 100m = 30m de CPU moyen par pod
```

- Si la consommation moyenne d'un pod dépasse **30m**, l'HPA augmente le nombre de replicas.
- Si elle redescend sous ce seuil, il le diminue.
- La `limit` à `200m` borne ce que chaque pod peut consommer, mais **n'empêche pas** l'HPA de
  créer d'autres pods : la charge est répartie sur plusieurs replicas.

---

## 3. `hpa-application.yml` — le HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-application
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-application
  minReplicas: 1
  maxReplicas: 5
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 0
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 30
```

### Explication

| Élément | Rôle |
|---|---|
| `apiVersion: autoscaling/v2` | Version actuelle de l'API HPA (support des métriques ressources). |
| `kind: HorizontalPodAutoscaler` | Objet qui ajuste automatiquement le nombre de replicas. |
| `metadata.name: mon-application` | Nom du HPA, identique au Deployment ciblé (comme demandé). |
| `scaleTargetRef` | Indique **quel objet** scaler : le Deployment `mon-application`. |
| `minReplicas: 1` | Ne descend jamais en dessous d'un pod. |
| `maxReplicas: 5` | Plafond de sécurité : au maximum 5 pods. |
| `behavior.scaleDown.stabilizationWindowSeconds: 0` | Réduit immédiatement après la baisse de charge (réglage de démonstration). |
| `metrics[].type: Resource` | On utilise une métrique de ressource (CPU). |
| `resource.name: cpu` | Métrique surveillée : le CPU. |
| `target.type: Utilization` | La cible est un **pourcentage de la request**. |
| `averageUtilization: 30` | Seuil : 30 % de `requests.cpu`, soit **30m** par pod en moyenne. |

### Formule utilisée par l'HPA

```text
replicas désirés = ceil( replicas actuels × (métrique actuelle / métrique cible) )
```

Exemple avec `requests.cpu = 100m` et une cible de 30 % (= 30m) :

| Consommation moyenne observée | Calcul | Replicas désirés |
|---|---|---|
| 20m (20 %) | `ceil(1 × 20/30)` | 1 |
| 35m (35 %) | `ceil(1 × 35/30)` | 2 |
| 70m (70 %) | `ceil(1 × 70/30)` | 3 |
| 150m (150 %) | `ceil(1 × 150/30)` | 5 (plafonné par `maxReplicas`) |

> **Astuce :** L'HPA ne crée pas les pods lui-même : il modifie le champ `replicas` du Deployment, et
> c'est le Deployment qui crée les pods. Le Service, s'il existe, répartit ensuite le trafic.

---

## 4. ❓ Résultat souhaité avant de poursuivre

Après avoir appliqué les deux fichiers, on doit observer :

1. **Au repos** : le Deployment `mon-application` a **1 pod**, et le HPA affiche une
   consommation faible, par exemple :

   ```text
   NAME              REFERENCE                    TARGETS       MINPODS   MAXPODS   REPLICAS
   mon-application   Deployment/mon-application   cpu: 0%/30%   1         5         1
   ```

2. **Sous charge** : la consommation CPU moyenne dépasse 30 % de la request (soit 30m). L'HPA
   calcule alors un nombre de replicas supérieur et **le Deployment crée un ou plusieurs pods
   supplémentaires**, jusqu'à 5 maximum.

3. **Après l'arrêt de la charge** : la consommation redescend sous le seuil. L'HPA ramène
   progressivement le Deployment à **1 pod**. Avec la fenêtre de stabilisation par défaut
   (~5 minutes), la descente est plus lente ; je l'ai mise à `0` pour rendre le test visible
   immédiatement.

4. **Pendant tout le processus** : le Service créé à l'étape 5 continue de répondre ; les
   nouveaux pods sont ajoutés automatiquement à ses endpoints.

En résumé : on veut un système qui **passe automatiquement de 1 à plusieurs pods quand la charge
augmente**, puis **revient à 1 pod quand la charge retombe**, sans intervention manuelle.

---

## 5. ❓ Exposer l'application localement sur le port 80

J'ai exposé le Deployment avec la commande demandée :

```bash
kubectl expose deployment mon-application --port=80 --type=LoadBalancer
```

Résultat :

```text
service/mon-application exposed
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
mon-application   LoadBalancer   10.101.247.57   <pending>     80:32292/TCP   1s
```

Sur Minikube, l'`EXTERNAL-IP` reste `<pending>` tant qu'on n'a pas lancé un tunnel
(`minikube tunnel`). Ce n'est pas bloquant pour le TP : le Service possède déjà une IP interne
et un `NodePort` (`32292`).

Pour ouvrir l'application dans un navigateur, j'ai utilisé un `port-forward` local :

```bash
kubectl port-forward svc/mon-application 8080:80
```

Puis <http://localhost:8080> :

```text
HTTP 200
<title>Welcome to nginx!</title>
```

📸 Capture : [`img/partie3-hpa-application.png`](img/partie3-hpa-application.png)

---

## 6. ❓ Vérifier que le HPA est bien en place

J'ai utilisé deux commandes complémentaires :

```bash
# Vue synthétique
kubectl get hpa mon-application

# Vue détaillée : métriques, comportement, conditions et événements
kubectl describe hpa mon-application
```

Exemple de sortie au repos :

```text
NAME              REFERENCE                    TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
mon-application   Deployment/mon-application   cpu: 0%/30%   1         5         1          8d
```

---

## 7. ❓ Signification des colonnes retournées

### `kubectl get hpa`

| Colonne | Signification |
|---|---|
| `NAME` | Nom du HPA. |
| `REFERENCE` | Objet ciblé, ici `Deployment/mon-application`. |
| `TARGETS` | Métrique **actuelle / cible**. `cpu: 35%/30%` signifie 35 % de la request CPU pour une cible de 30 %. `<unknown>` apparaît si metrics-server ne répond pas encore. |
| `MINPODS` | Nombre minimal de replicas. |
| `MAXPODS` | Nombre maximal de replicas. |
| `REPLICAS` | Nombre actuel de replicas du Deployment. |
| `AGE` | Âge du HPA. |

### `kubectl describe hpa`

La sortie détaillée ajoute :

- **Metrics** : `current / target`, par exemple `0% (0) / 30%`.
- **Behavior** : politiques de scale-up et scale-down (fenêtre de stabilisation, pas de
  progression).
- **Deployment pods** : `1 current / 1 desired`.
- **Conditions** :
  - `AbleToScale` : le HPA peut-il lire et modifier la cible ?
  - `ScalingActive` : la métrique est-elle valide ?
  - `ScalingLimited` : le résultat est-il borné par `minReplicas` / `maxReplicas` ?
  - `ScaledToZero` : le HPA a-t-il réduit à zéro ?
- **Events** : historique des changements, par exemple
  `New size: 2; reason: cpu resource utilization (percentage of request) above target`.

---

## 8. ❓ Test de charge et capture

J'ai créé un pod `busybox` qui interroge le Service en boucle :

```bash
kubectl run simulation-utilisateur \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c 'while true; do wget -q -O- http://mon-application.default.svc.cluster.local > /dev/null; done'
```

Avec une seule boucle, le CPU tournait autour de **23 %**. Avec 3 pods de charge, il est monté à
**32 %**, juste au-dessus de la cible de 30 %, mais l'HPA applique une tolérance de 10 % : à
32 %, il considère qu'on est encore « proche » de la cible et il ne scale pas. J'ai donc lancé
**6 pods de charge** pour dépasser franchement le seuil :

```bash
for n in 1 2 3 4 5 6; do
  kubectl run "verif-charge-$n" \
    --image=busybox:1.36 \
    --restart=Never \
    -- /bin/sh -c 'while true; do wget -q -O- http://mon-application.default.svc.cluster.local > /dev/null; done'
done
```

### Chronologie observée

```text
===== 01:08:30 =====
NAME              REFERENCE                    TARGETS        MINPODS   MAXPODS   REPLICAS
mon-application   Deployment/mon-application   cpu: 49%/30%   1         5         1

===== 01:08:50 =====
mon-application   Deployment/mon-application   cpu: 49%/30%   1         5         2

===== 01:09:56 =====
mon-application   Deployment/mon-application   cpu: 35%/30%   1         5         3
```

L'HPA est passé de **1 → 2 → 3 pods** : le scale-up fonctionne.

Après suppression des pods de charge :

```bash
kubectl delete pod verif-charge-1 verif-charge-2 verif-charge-3 \
  verif-charge-4 verif-charge-5 verif-charge-6
```

```text
===== 01:11:30 =====
mon-application   Deployment/mon-application   cpu: 14%/30%   1         5         3

===== 01:12:55 =====
mon-application   Deployment/mon-application   cpu: 14%/30%   1         5         2

===== 01:13:05 =====
mon-application   Deployment/mon-application   cpu: 16%/30%   1         5         1

===== 01:13:35 =====
mon-application   Deployment/mon-application   cpu:  0%/30%   1         5         1
```

Puis, une fois les métriques redescendues, retour à **1 pod**. Les événements du HPA résument
parfaitement le scénario :

```text
Normal  SuccessfulRescale  5m50s (x2 over 6h55m)  New size: 2; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  4m50s (x2 over 6h54m)  New size: 3; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  110s (x2 over 6h58m)   New size: 2; reason: All metrics below target
Normal  SuccessfulRescale  95s (x3 over 6h57m)    New size: 1; reason: All metrics below target
```

📸 Capture de l'application exposée : [`img/partie3-hpa-application.png`](img/partie3-hpa-application.png)  
📸 Capture du scale-up / scale-down : [`img/partie3-hpa-scaling.png`](img/partie3-hpa-scaling.png)

---

## 9. ❓ L'HPA fonctionne-t-il comme souhaité ? Ajustements

**Oui.** Le HPA :

- lit bien les métriques CPU via metrics-server (`TARGETS` n'est pas `<unknown>`) ;
- monte de **1 à 3 pods** quand la charge dépasse la cible ;
- redescend à **1 pod** quand la charge retombe ;
- respecte `minReplicas: 1` et `maxReplicas: 5`.

### Ajustements effectués

1. **Cible passée de 50 % à 30 %** : une boucle `wget` unique plafonnait à ~50m, soit exactement
   50 % de la request. L'HPA ne réagit que si la valeur **dépasse** la cible. Avec 30 %, la
   marge est suffisante pour déclencher le scale-up de façon fiable.
2. **Ajout de plusieurs pods de charge** : avec 3 pods, le CPU tournait autour de 32 %, dans la
   tolérance de 10 % de l'HPA. Avec 6 pods, il est monté à 49 %, ce qui déclenche le scale-up de
   manière nette.
3. **`stabilizationWindowSeconds: 0`** sur le scale-down : réglage **de démonstration** pour voir
   la descente immédiatement. En production, on garde la fenêtre par défaut (~5 min) pour éviter
   les oscillations (flapping).

### Remarque production

La cible de 30 % est volontairement basse pour un TP local. En production, on viserait plutôt
60–80 % d'utilisation CPU, avec des requests/limits mesurées sur la charge réelle, et un
`PodDisruptionBudget` pour garantir la disponibilité pendant les changements de replicas.

---

## 10. Récapitulatif des commandes

```bash
# Pré-requis
minikube addons enable metrics-server
kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes

# Déploiement de l'application et du HPA
kubectl apply -f TP/hpa/application.yml
kubectl apply -f TP/hpa/hpa-application.yml

# Exposition
kubectl expose deployment mon-application --port=80 --type=LoadBalancer
kubectl get svc mon-application
kubectl port-forward svc/mon-application 8080:80

# Vérifications
kubectl get deployment mon-application
kubectl get pods -l app=mon-application
kubectl top pods -l app=mon-application
kubectl get hpa mon-application
kubectl describe hpa mon-application

# Charge (6 pods pour dépasser franchement la cible)
for n in 1 2 3 4 5 6; do
  kubectl run "verif-charge-$n" --image=busybox:1.36 --restart=Never -- /bin/sh -c \
    'while true; do wget -q -O- http://mon-application.default.svc.cluster.local > /dev/null; done'
done
kubectl get hpa mon-application

# Fin de charge
kubectl delete pod verif-charge-1 verif-charge-2 verif-charge-3 \
  verif-charge-4 verif-charge-5 verif-charge-6
```

---

## 11. Livrables de cette section

- [x] `TP/hpa/application.yml`
- [x] `TP/hpa/hpa-application.yml`
- [x] Requests/limits définies et cohérentes avec la cible du HPA
- [x] Service `LoadBalancer` exposé sur le port 80 (étape 5)
- [x] HPA vérifié avec `kubectl get hpa` et `kubectl describe hpa` (étape 6)
- [x] Colonnes expliquées (étape 7)
- [x] Test de charge réalisé, scale-up 1 → 3 puis retour à 1 (étape 8)
- [x] Réponse à la question 9 et ajustements documentés
- [x] Captures :
  - [`img/partie3-hpa-application.png`](img/partie3-hpa-application.png)
  - [`img/partie3-hpa-scaling.png`](img/partie3-hpa-scaling.png)
