# TP Kubernetes — Partie 5 : Rendre l'application « production-ready »

> Application : `paulbouwer/hello-kubernetes`
> Namespace : `prod-app`
> Fichiers : [`TP/prod-app/namespace.yml`](../TP/prod-app/namespace.yml),
> [`TP/prod-app/app-deployment.yml`](../TP/prod-app/app-deployment.yml),
> [`TP/prod-app/app-service.yml`](../TP/prod-app/app-service.yml)

J'ai construit cette partie progressivement : d'abord le Deployment et le Service, puis les
requests/limits, les probes, le rolling update et le `preStop`. J'ai gardé les captures et les
sorties de commandes telles quelles, y compris le passage où l'image ne répond pas sur
`/healthz` et `/ready` : c'est un point que j'ai dû adapter pour que le TP reste fonctionnel.

---

## Partie 1 — Déploiement initial

### 1.1 Namespace, Deployment et Service

J'ai créé le namespace `prod-app` :

```bash
kubectl apply -f TP/prod-app/namespace.yml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod-app
```

Le Deployment démarre avec **3 réplicas** de l'image demandée :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
  namespace: prod-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
        - name: hello-kubernetes
          image: paulbouwer/hello-kubernetes:1.10
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
```

Le Service est de type `ClusterIP` et expose le port `8080` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes
  namespace: prod-app
spec:
  type: ClusterIP
  selector:
    app: hello-kubernetes
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
```

Déploiement :

```bash
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl apply -f TP/prod-app/app-service.yml
kubectl rollout status deployment/hello-kubernetes -n prod-app
```

Résultat :

```text
deployment "hello-kubernetes" successfully rolled out

NAME                                READY   STATUS    RESTARTS   AGE
hello-kubernetes-5c45f7856c-6wbsl   1/1     Running   0          7s
hello-kubernetes-5c45f7856c-fgf7x   1/1     Running   0          7s
hello-kubernetes-5c45f7856c-nc9vb   1/1     Running   0          7s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hello-kubernetes   ClusterIP   10.111.237.212   <none>        8080/TCP   7s
```

### 1.2 Accès depuis le navigateur

J'ai utilisé un `port-forward` (port local `8086`, car `8080` était déjà occupé par un autre TP) :

```bash
kubectl port-forward -n prod-app svc/hello-kubernetes 8086:8080
```

Puis <http://localhost:8086> :

📸 Capture : [`img/partie5-hello-kubernetes.png`](img/partie5-hello-kubernetes.png)

### 1.3 ❓ Que remarque-t-on sur le « Pod Name » à chaque rafraîchissement ?

Le nom du pod **change** d'une requête à l'autre : l'application affiche le hostname du pod qui
a traité la requête, et le Service répartit les connexions entre les **3 réplicas**.

J'ai vérifié la répartition depuis un pod `busybox` placé dans le cluster (le `port-forward`
vers un Service, lui, reste attaché à un seul pod backend, d'où un nom parfois fixe dans le
navigateur) :

```bash
kubectl run test-loadbalancing --rm -i --restart=Never -n prod-app \
  --image=busybox:1.36 -- sh -c \
  'for i in $(seq 1 15); do wget -q -O- http://hello-kubernetes:8080/ \
   | grep -oE "hello-kubernetes-[a-z0-9]+-[a-z0-9]+" | head -1; done'
```

```text
hello-kubernetes-5c45f7856c-nc9vb
hello-kubernetes-5c45f7856c-fgf7x
hello-kubernetes-5c45f7856c-nc9vb
hello-kubernetes-5c45f7856c-6wbsl
hello-kubernetes-5c45f7856c-nc9vb
hello-kubernetes-5c45f7856c-fgf7x
hello-kubernetes-5c45f7856c-fgf7x
hello-kubernetes-5c45f7856c-fgf7x
hello-kubernetes-5c45f7856c-6wbsl
hello-kubernetes-5c45f7856c-6wbsl
hello-kubernetes-5c45f7856c-6wbsl
hello-kubernetes-5c45f7856c-fgf7x
hello-kubernetes-5c45f7856c-fgf7x
hello-kubernetes-5c45f7856c-nc9vb
```

**Pourquoi ?** Le Service possède une IP virtuelle stable. `kube-proxy` programme des règles
(iptables/IPVS) sur chaque nœud pour rediriger le trafic vers l'un des pods sélectionnés par le
label `app: hello-kubernetes`. Chaque nouvelle connexion peut donc atterrir sur un pod différent.
C'est ce qui permet la répartition de charge et la tolérance à la panne d'un pod.

---

## Partie 1.2 — Requests, limits et QoS

### Configuration ajoutée

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

Application :

```bash
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl rollout status deployment/hello-kubernetes -n prod-app
```

### ❓ À quelle classe QoS appartient le pod ?

```bash
kubectl get pods -n prod-app \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[0].ready,QOS:.status.qosClass'
```

```text
NAME                                READY   QOS
hello-kubernetes-59c94dd874-5c2mw   true    Burstable
hello-kubernetes-59c94dd874-bzn4p   true    Burstable
hello-kubernetes-59c94dd874-dflg8   true    Burstable
```

**Réponse : `Burstable`.**

Kubernetes attribue la classe QoS en fonction des requests et limits :

| Classe | Condition |
|---|---|
| `Guaranteed` | Chaque conteneur a des requests **et** des limits pour CPU et mémoire, et `requests == limits`. |
| `Burstable` | Au moins une request ou une limit est définie, mais les valeurs ne sont pas toutes égales. |
| `BestEffort` | Aucune request ni limit n'est définie. |

Ici, `requests.cpu = 100m` mais `limits.cpu = 200m` (et idem pour la mémoire) : les valeurs sont
différentes, donc le pod est **Burstable**.

### Pourquoi viser au minimum `Burstable` en production ?

- Un pod `BestEffort` n'a **aucune garantie** de ressources : il est le premier candidat à
  l'éviction quand le nœud est sous pression (mémoire/CPU), et le Scheduler peut le placer
  n'importe où.
- Un pod `Burstable` réserve au moins ses `requests` : le Scheduler lui garantit ces ressources
  et il est moins susceptible d'être évincé qu'un `BestEffort`.
- La classe `Guaranteed` est encore plus forte (requests = limits), mais elle est plus rigide et
  peut gaspiller des ressources. Pour une application de production, `Burstable` avec des
  requests réalistes est un bon compromis ; `Guaranteed` est intéressant pour les services
  critiques et prévisibles.

---

## Partie 2 — Probes

### 2.1 `livenessProbe`

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

### ❓ Que se passe-t-il en cas de deadlock ?

Si le processus est bloqué (deadlock) sans crasher, le conteneur reste `Running` mais ne répond
plus. La `livenessProbe` détecte ce cas : après `failureThreshold` échecs consécutifs, le kubelet
**redémarre le conteneur** automatiquement. C'est ce qui permet de récupérer d'un blocage sans
intervention manuelle.

### 2.2 `readinessProbe`

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

> **Remarque importante sur l'image fournie.** Le sujet indique que l'application expose
> `/healthz` et `/ready`. En réalité, l'image `paulbouwer/hello-kubernetes:1.10` (et `1.10.1`)
> renvoie **404** sur ces deux chemins. Je l'ai vérifié dans les logs :
>
> ```text
> GET /healthz -> 404
> GET /ready   -> 404
> ```
>
> Pour que le TP reste fonctionnel, j'ai utilisé `/` comme chemin de sonde (qui renvoie 200),
> en conservant les mêmes mécanismes et le `initialDelaySeconds: 10` sur la readiness. Si
> l'image exposait réellement `/healthz` et `/ready`, il suffirait de remplacer les chemins.

Application et observation :

```bash
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl get pods -n prod-app -l app=hello-kubernetes -w
```

```text
NAME                                READY   STATUS    RESTARTS   AGE
hello-kubernetes-55c77f974b-p7t8k   1/1     Running   0          17s
hello-kubernetes-55c77f974b-r8rlv   0/1     Running   0          1s
hello-kubernetes-55c77f974b-tmfn5   1/1     Running   0          28s
hello-kubernetes-55c77f974b-r8rlv   1/1     Running   0          11s
```

Le pod reste **0/1** pendant environ 10 secondes, puis passe à **1/1** : c'est exactement
l'effet de `initialDelaySeconds: 10`.

Extrait de `kubectl describe pod ... -n prod-app` :

```text
Liveness:     http-get http://:8080/ delay=5s timeout=1s period=10s successThreshold=1 failureThreshold=3
Readiness:    http-get http://:8080/ delay=10s timeout=1s period=5s successThreshold=1 failureThreshold=3

Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
```

📸 Capture : [`img/partie5-readiness-events.png`](img/partie5-readiness-events.png)

### ❓ Différence de conséquence entre liveness et readiness

| Sonde | En cas d'échec | Conséquence |
|---|---|---|
| `livenessProbe` | Le conteneur ne répond plus / est bloqué. | Le kubelet **redémarre le conteneur** (le pod reste sur le même nœud). |
| `readinessProbe` | Le conteneur est vivant mais pas prêt (démarrage, surcharge, dépendance indisponible). | Le pod est **retiré des endpoints du Service** : il ne reçoit plus de trafic, mais **n'est pas redémarré**. |

Dans le second cas, c'est le **Service** (via la mise à jour des `EndpointSlices` par le
contrôleur d'endpoints) puis **`kube-proxy`** qui cessent d'envoyer du trafic au pod. Dès que la
readiness redevient verte, le pod est réintégré automatiquement.

---

## Partie 3 — Déploiement sans interruption et arrêt propre

### 3.1 RollingUpdate

J'ai changé l'image de `1.10` vers `1.10.1` :

```yaml
image: paulbouwer/hello-kubernetes:1.10.1
```

```bash
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl get pods -n prod-app -l app=hello-kubernetes -w
```

```text
hello-kubernetes-859cbb6897-mvggp   0/1     ContainerCreating   0          0s
hello-kubernetes-859cbb6897-mvggp   0/1     Running             0          1s
hello-kubernetes-859cbb6897-mvggp   1/1     Running             0          12s
hello-kubernetes-55c77f974b-r8rlv   1/1     Terminating         0          56s
hello-kubernetes-859cbb6897-wdgcr   0/1     ContainerCreating   0          0s
hello-kubernetes-859cbb6897-wdgcr   0/1     Running             0          0s
hello-kubernetes-859cbb6897-wdgcr   1/1     Running             0          11s
hello-kubernetes-55c77f974b-tmfn5   1/1     Terminating         0          94s
hello-kubernetes-859cbb6897-ls2v9   0/1     ContainerCreating   0          0s
hello-kubernetes-859cbb6897-ls2v9   0/1     Running             0          0s
hello-kubernetes-859cbb6897-ls2v9   1/1     Running             0          11s
hello-kubernetes-55c77f974b-p7t8k   1/1     Terminating         0          94s
```

On voit bien le remplacement **pod par pod** : un nouveau pod est créé, il passe `0/1` puis
`1/1` après sa readiness, et seulement ensuite un ancien pod passe `Terminating`.

📸 Capture : [`img/partie5-rolling-update.png`](img/partie5-rolling-update.png)

### ❓ Rôle des readinessProbes dans un RollingUpdate sans interruption

Les readiness probes sont **essentielles** : un nouveau pod n'est ajouté aux endpoints du Service
qu'après avoir passé sa readiness. Tant qu'il n'est pas prêt, il ne reçoit aucun trafic. Les
anciens pods restent disponibles et continuent de servir les requêtes jusqu'à ce que les
nouveaux soient prêts. Combiné aux paramètres `maxUnavailable` et `maxSurge` du Deployment, cela
évite les erreurs `503` : il y a toujours au moins un pod prêt à répondre.

### 3.2 Arrêt gracieux (`preStop`)

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

Application puis suppression manuelle d'un pod :

```bash
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl delete pod hello-kubernetes-75d6cc444f-wzc4k -n prod-app --wait=false
kubectl get pod hello-kubernetes-75d6cc444f-wzc4k -n prod-app
```

```text
--- 18:32:59 ---
hello-kubernetes-75d6cc444f-wzc4k   1/1     Terminating   0          37s
--- 18:33:01 ---
hello-kubernetes-75d6cc444f-wzc4k   1/1     Terminating   0          39s
--- 18:33:03 ---
hello-kubernetes-75d6cc444f-wzc4k   1/1     Terminating   0          41s
--- 18:33:05 ---
hello-kubernetes-75d6cc444f-wzc4k   1/1     Terminating   0          44s
--- 18:33:08 ---
hello-kubernetes-75d6cc444f-wzc4k   1/1     Terminating   0          46s
--- 18:33:10 ---
hello-kubernetes-75d6cc444f-wzc4k   1/1     Terminating   0          48s
--- 18:33:12 ---
Error from server (NotFound): pods "hello-kubernetes-75d6cc444f-wzc4k" not found
```

Le pod reste en `Terminating` environ **10 à 13 secondes** : c'est le hook `preStop` qui
s'exécute avant l'arrêt effectif du conteneur.

📸 Capture : [`img/partie5-prestop.png`](img/partie5-prestop.png)

### ❓ À quoi sert le `preStop` ?

Le hook `preStop` laisse à l'application un **délai de grâce** avant que le conteneur ne soit
arrêté. Dans un vrai scénario, l'application ne se contente pas de dormir : elle peut :

- arrêter d'accepter de **nouvelles requêtes** ;
- **terminer les requêtes en cours** (in-flight) ;
- fermer proprement les **connexions aux bases de données** ;
- **vider les buffers** et envoyer les derniers logs/métriques ;
- se **désenregistrer** d'un annuaire de services ou d'un load balancer externe.

Cela évite de couper brutalement des requêtes utilisateur et de laisser des ressources dans un
état incohérent.

---

## Partie 4 — Débogage avec des conteneurs éphémères

### 4.1 Le problème : `curl` absent de l'image

```bash
kubectl exec -n prod-app hello-kubernetes-75d6cc444f-7vms2 -- curl localhost:8080
```

```text
error: Internal error occurred: ... exec: "curl": executable file not found in $PATH
```

L'image de production est minimale : elle ne contient ni `curl`, ni `ping`, ni `vim`.

### 4.2 La solution : `kubectl debug`

```bash
kubectl debug -n prod-app hello-kubernetes-75d6cc444f-7vms2 \
  --image=busybox:1.36 --target=hello-kubernetes -- wget -O - localhost:8080
```

```text
Targeting container "hello-kubernetes". ...
Defaulting debug container name to debugger-c5rdk.
```

Le conteneur éphémère partage le réseau du pod : `localhost:8080` pointe donc bien sur
l'application. J'ai récupéré sa sortie via les logs :

```bash
kubectl logs hello-kubernetes-75d6cc444f-7vms2 -n prod-app -c debugger-c5rdk
```

```text
Connecting to localhost:8080 (127.0.0.1:8080)
writing to stdout
-                    100% |********************************|  1559  0:00:00 ETA
written to stdout
<!DOCTYPE html>
<html>
<head>
    <title>Hello Kubernetes!</title>
    ...
</head>
<body>
    ...
    <tr>
      <th>pod:</th>
      <td>hello-kubernetes-75d6cc444f-7vms2</td>
    </tr>
    ...
</body>
</html>
```

📸 Capture : [`img/partie5-debug.png`](img/partie5-debug.png)

### ❓ Pourquoi cette approche est-elle meilleure que d'inclure les outils dans l'image ?

- **Surface d'attaque réduite** : moins de binaires dans l'image = moins de vulnérabilités
  potentielles (CVE) et moins de privilèges exploitables.
- **Image plus petite et plus rapide** à construire, tirer et démarrer.
- **Pas de pollution de l'image de production** : les outils de debug ne sont pas nécessaires au
  fonctionnement normal et ne doivent pas être livrés aux utilisateurs.
- **Débogage sans rebuild ni redéploiement** : le conteneur éphémère est créé à la demande,
  partage les namespaces du pod cible (réseau, processus selon la configuration) et disparaît
  avec le pod.
- **Traçabilité** : l'ajout d'un conteneur de debug est visible dans l'API Kubernetes et peut
  être soumis à RBAC/audit, contrairement à des outils embarqués en permanence.

---

## Récapitulatif des commandes

```bash
# Namespace et déploiement
kubectl apply -f TP/prod-app/namespace.yml
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl apply -f TP/prod-app/app-service.yml
kubectl rollout status deployment/hello-kubernetes -n prod-app

# Accès local
kubectl port-forward -n prod-app svc/hello-kubernetes 8086:8080

# Vérifications
kubectl get pods -n prod-app -o wide
kubectl get svc -n prod-app
kubectl get pods -n prod-app \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[0].ready,QOS:.status.qosClass'

# Probes
kubectl describe pod <pod> -n prod-app
kubectl get pods -n prod-app -l app=hello-kubernetes -w

# Rolling update
kubectl apply -f TP/prod-app/app-deployment.yml
kubectl get pods -n prod-app -l app=hello-kubernetes -w

# Arrêt gracieux
kubectl delete pod <pod> -n prod-app --wait=false
kubectl get pod <pod> -n prod-app

# Débogage
kubectl exec -n prod-app <pod> -- curl localhost:8080
kubectl debug -n prod-app <pod> --image=busybox:1.36 --target=hello-kubernetes -- wget -O - localhost:8080
kubectl logs <pod> -n prod-app -c <debug-container>
```

---

## Livrables

- [x] `TP/prod-app/namespace.yml`
- [x] `TP/prod-app/app-deployment.yml`
- [x] `TP/prod-app/app-service.yml`
- [x] Deployment 3 réplicas + Service ClusterIP
- [x] Requests/limits et justification de la classe QoS `Burstable`
- [x] `livenessProbe` et `readinessProbe` (avec `initialDelaySeconds: 10`)
- [x] RollingUpdate de `1.10` vers `1.10.1` observé pod par pod
- [x] Hook `preStop` et arrêt gracieux observé
- [x] Débogage par conteneur éphémère `busybox`
- [x] Captures :
  - [`img/partie5-hello-kubernetes.png`](img/partie5-hello-kubernetes.png)
  - [`img/partie5-readiness-events.png`](img/partie5-readiness-events.png)
  - [`img/partie5-rolling-update.png`](img/partie5-rolling-update.png)
  - [`img/partie5-prestop.png`](img/partie5-prestop.png)
  - [`img/partie5-debug.png`](img/partie5-debug.png)

---

## Ce que j'ai retenu

C'est la partie où j'ai le plus manipulé les probes et les stratégies de déploiement. Le
`RollingUpdate` est vraiment rassurant quand les readiness probes sont bien réglées : on voit
les nouveaux pods passer `0/1` puis `1/1`, et les anciens ne partent qu'après. Sans readiness,
Kubernetes enverrait du trafic à un pod qui n'est pas encore prêt.

Le `preStop` est un détail qu'on oublie vite : un `sleep 10` suffit pour le TP, mais dans une
vraie application il faut surtout arrêter d'accepter de nouvelles requêtes et finir celles en
cours. J'ai aussi trouvé `kubectl debug` très pratique : on garde une image de production
minimale et on ajoute un conteneur `busybox` juste le temps du diagnostic.

Le point qui m'a surpris, c'est que l'image `hello-kubernetes:1.10` ne renvoie pas 200 sur
`/healthz` et `/ready` (404). J'ai adapté les sondes sur `/` et je l'ai noté dans le compte
rendu, parce que ce n'est pas ce que le sujet annonçait.
