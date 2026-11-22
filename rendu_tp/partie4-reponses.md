# TP Kubernetes — Partie 4 : QCM

> Réponses au QCM, avec une justification courte pour chaque question.
> J'ai gardé le format du sujet et marqué la bonne réponse d'un `X`.

---

## Réponses rapides

| Question | Réponse | Thème |
|---|---|---|
| 1 | **B** | Pod |
| 2 | **A et B** | Namespace |
| 3 | **A** | Communication avec l'API Server |
| 4 | **B** | ConfigMap |
| 5 | **B** | Job planifié |
| 6 | **B** | Deployment |
| 7 | **B** | Service / kube-proxy |
| 8 | **A** | Logs |
| 9 | **A** | Échec d'un Pod |
| 10 | **C** | ConfigMap vs Secret |
| 11 | **B** | Namespace |
| 12 | **C** | Volume persistant |
| 13 | **C** | API Server |
| 14 | **B** | Sécurité et isolation |
| 15 | **A** | Requests / limits / QoS |
| Bonus | **C** | ConfigMaps et Secrets |

---

## Question 1 — Qu'est-ce qu'un Pod ?

- A. Un conteneur unique
- **X B. Un groupe de conteneurs agissant comme une unité logique**
- C. Une méthode de gestion des secrets

**Réponse : B.** Un Pod est la plus petite unité déployable de Kubernetes. Il peut contenir un
ou plusieurs conteneurs qui partagent le réseau, le stockage et le cycle de vie. En pratique, on
y met souvent un conteneur principal et éventuellement des conteneurs « sidecar ».

---

## Question 2 — Quelle commande crée un Namespace ?

- **X A. `kubectl create ns <nom-du-namespace>`**
- **X B. `kubectl create namespace <nom-du-namespace>`**
- C. `kubectl new namespace <nom-du-namespace>`

**Réponse : A et B.** `ns` est l'alias officiel de `namespace` dans `kubectl`. Les deux commandes
sont donc valides. La commande `kubectl new namespace` n'existe pas.

---

## Question 3 — Quel composant communique avec l'API Server ?

- **X A. kubelet**
- B. etcd
- C. kube-proxy

**Réponse : A.** Le `kubelet` est l'agent présent sur chaque nœud. Il interroge l'API Server pour
connaître les Pods à exécuter, remonte l'état du nœud et des conteneurs, et communique les
événements. `etcd` est la base de données clé/valeur du cluster ; `kube-proxy` gère les règles
réseau des Services.

---

## Question 4 — Qu'est-ce qu'un ConfigMap ?

- A. Un type de service
- **X B. Une ressource pour stocker des données de configuration non sensibles**
- C. Un mécanisme de mise à l'échelle horizontale

**Réponse : B.** Un ConfigMap stocke de la configuration sous forme de paires clé/valeur
(variables d'environnement, fichiers de configuration…). Il ne doit pas contenir de données
sensibles : pour cela, on utilise un Secret.

---

## Question 5 — Comment exécuter un Job planifié ?

- A. En utilisant un Job
- **X B. En utilisant un CronJob**
- C. En utilisant un DaemonSet

**Réponse : B.** Un `Job` exécute une tâche ponctuelle jusqu'à son succès. Un `CronJob` crée des
Jobs selon une planification de type cron (par exemple toutes les 5 minutes). Un `DaemonSet`
exécute un Pod sur chaque nœud.

---

## Question 6 — Qu'est-ce qu'un Deployment ?

- A. Une méthode pour déployer des secrets dans un cluster
- **X B. Un contrôleur responsable de la gestion des mises à jour de vos pods**
- C. Un conteneur unique géré par kubelet

**Réponse : B.** Un Deployment gère un ReplicaSet et donc un ensemble de Pods interchangeables.
Il permet de définir le nombre de replicas, de réaliser des mises à jour progressives
(rolling update) et de revenir en arrière (rollback).

---

## Question 7 — Équilibrage de charge et basculement entre les Pods d'un Service

- A. kube-scheduler
- **X B. kube-proxy**
- C. kube-controller-manager

**Réponse : B.** `kube-proxy` programme les règles de réseau (iptables/IPVS) sur chaque nœud pour
répartir le trafic d'un Service vers les Pods disponibles. Le `kube-scheduler` choisit le nœud
d'un Pod ; le `kube-controller-manager` exécute les contrôleurs internes.

---

## Question 8 — Afficher les journaux d'un Pod

- **X A. `kubectl logs <pod-name>`**
- B. `kubectl view logs <pod-name>`
- C. `kubectl get logs <pod-name>`

**Réponse : A.** `kubectl logs` affiche la sortie standard du conteneur. On peut ajouter `-f`
pour suivre en temps réel, `-c <conteneur>` pour choisir un conteneur d'un Pod multi-conteneurs,
ou `--previous` pour voir les logs de l'instance précédente.

---

## Question 9 — Que se passe-t-il lorsqu'un Pod échoue ?

- **X A. Il est recréé par une réplication**
- B. Il reste en état d'échec
- C. Il est déplacé vers un autre nœud

**Réponse : A.** Si le Pod est géré par un contrôleur (Deployment, ReplicaSet, StatefulSet…),
celui-ci détecte l'échec et recrée un Pod pour maintenir le nombre de replicas désiré. Un Pod
« nu » (créé directement sans contrôleur) reste, lui, en échec.

---

## Question 10 — Différence entre ConfigMap et Secret

- A. Aucun, ils sont identiques
- B. ConfigMap est pour des données sensibles
- **X C. Secret est pour des données sensibles**

**Réponse : C.** Les deux stockent des paires clé/valeur, mais un Secret est destiné aux données
sensibles (mots de passe, tokens, certificats). Les valeurs sont encodées en base64 dans etcd ;
ce n'est pas du chiffrement à proprement parler.

---

## Question 11 — Qu'est-ce qu'un Namespace ?

- A. Un type de Pod
- **X B. Une méthode de partitionnement du cluster en plusieurs environnements virtuels**
- C. Une unité de stockage persistent

**Réponse : B.** Un Namespace permet de regrouper et d'isoler logiquement des ressources
(développement, test, production, équipes…). Les noms de ressources doivent être uniques à
l'intérieur d'un Namespace, mais peuvent être réutilisés dans un autre.

---

## Question 12 — Comment réclamer un volume ?

- A. PersistentClaim
- B. AvailableVolume
- **X C. PersistentVolumeClaim**

**Réponse : C.** Un `PersistentVolumeClaim` (PVC) est une demande de stockage formulée par une
application. Kubernetes associe ce PVC à un `PersistentVolume` (PV) disponible selon la taille,
le mode d'accès et la classe de stockage.

---

## Question 13 — Rôle de l'API Server

- A. Il stocke l'état souhaité du cluster et assure la persistance des données.
- B. Il décide quel Pod doit être programmé sur quel nœud à partir des ressources disponibles.
- **X C. Il expose l'API RESTful de Kubernetes et agit en tant que point d'entrée pour toutes les opérations d'administration du cluster.**

**Réponse : C.** L'API Server est le point d'entrée du plan de contrôle. Il valide les requêtes,
applique l'authentification et les autorisations, puis met à jour l'état dans etcd. C'est `etcd`
qui stocke les données (A), et le `kube-scheduler` qui choisit le nœud (B).

---

## Question 14 — Sécurité et isolation dans un cluster partagé

- A. En utilisant uniquement des firewalls pour séparer le trafic réseau entre les Pods.
- **X B. En appliquant le Role-Based Access Control (RBAC), les Pod Security Policies (PSP), et les Network Policies pour gérer les autorisations et l'isolation des Pods.**
- C. En exécutant tous les Pods dans le même espace de noms pour une meilleure coordination.

**Réponse : B.** Kubernetes combine plusieurs mécanismes : RBAC pour les autorisations,
Pod Security Standards/PSP pour les privilèges des Pods, Network Policies pour limiter les flux
réseau, et les Namespaces pour l'isolation logique. Un firewall seul ne suffit pas.

---

## Question 15 — Impact de requests/limits mal définies

- **X A. Si les requests sont trop basses, le Scheduler peut placer trop de Pods sur un nœud, entraînant une surallocation, tandis que des limits trop élevées peuvent provoquer un throttling intensif des applications.**
- B. Les requests et limits n'ont pas d'impact sur le Scheduler, ils sont uniquement utilisés pour la facturation des ressources.
- C. Des requests élevées garantissent la priorité dans l'allocation des ressources mais n'affectent pas le QoS des Pods.

**Réponse : A.** Les `requests` servent au Scheduler pour choisir un nœud disposant d'assez de
ressources. Trop basses, elles provoquent une surallocation et des évictions en cas de pression.
Les `limits` bornent la consommation : trop basses, elles entraînent du throttling CPU ou des
OOMKill mémoire ; trop élevées, elles laissent un Pod consommer au détriment des autres. Les
requests/limits déterminent aussi la classe QoS : `Guaranteed`, `Burstable` ou `BestEffort`.

---

## Bonus — Affirmation correcte sur ConfigMaps et Secrets

- A. Les ConfigMaps et les Secrets ne peuvent jamais être utilisés pour stocker des informations sensibles, car ils sont toujours exposés en texte clair dans le cluster.
- B. Il n'est pas possible de monter une ConfigMap ou un Secret en tant que volume dans les Pods.
- **X C. Les Secrets sont stockés de manière encodée en base64 dans etcd, mais ils ne sont pas chiffrés par défaut.**

**Réponse : C.** Les Secrets sont encodés en base64, ce qui n'est **pas** du chiffrement. Pour
les protéger réellement, il faut activer le chiffrement au repos dans etcd, utiliser RBAC, et
éventuellement un KMS externe. Les ConfigMaps et Secrets peuvent être montés en volumes (B est
faux), et les Secrets servent bien aux données sensibles (A est faux).

---

## Récapitulatif

- 15 questions + 1 bonus traitées.
- Toutes les réponses sont justifiées.
- Les points ambigus (Q3, Q9, Q15) sont précisés dans les explications.

---

## Ce que j'ai retenu

Les questions 3, 9 et 15 sont les plus ambiguës. Pour la 3, « communiquer avec l'API Server »
peut faire penser à etcd ou kube-proxy, mais c'est bien le kubelet qui dialogue avec l'API pour
récupérer les Pods à lancer. Pour la 9, tout dépend si le Pod est géré par un contrôleur ou non :
un Pod « nu » reste en échec, un Pod de Deployment est recréé.

La question 15 m'a obligé à revoir la différence entre `requests` (utilisées par le Scheduler) et
`limits` (plafond appliqué par le kubelet). C'est aussi ce qui détermine la classe QoS, donc ça
recoupe directement la partie 5.
