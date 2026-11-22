# TP Kubernetes — Partie 3 : CronJob

> J'ai traité ici la section **CronJob** de la partie 3.
> Le fichier YAML est dans [`TP/cronjob/cronjob.yml`](../TP/cronjob/cronjob.yml).

---

## 1. Fichier `cronjob.yml`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: date
spec:
  schedule: "*/5 * * * *"
  timeZone: "Europe/Paris"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: date
              image: busybox:1.36
              command: ["date"]
          restartPolicy: OnFailure
```

---

## 2. Explication détaillée

| Élément | Rôle |
|---|---|
| `apiVersion: batch/v1` | API des objets `Job` et `CronJob`. |
| `kind: CronJob` | Ressource qui crée automatiquement des `Job` selon une planification. |
| `metadata.name: date` | Nom du CronJob, demandé par le sujet. |
| `schedule: "*/5 * * * *"` | Exécution **toutes les 5 minutes**. |
| `timeZone: "Europe/Paris"` | Planification en heure de Paris (au lieu de l'heure UTC du controller). |
| `concurrencyPolicy: Forbid` | Interdit deux exécutions simultanées si la précédente n'est pas terminée. |
| `successfulJobsHistoryLimit: 3` | Conserve les 3 derniers Jobs réussis (et leurs logs). |
| `failedJobsHistoryLimit: 1` | Conserve le dernier Job échoué pour le diagnostic. |
| `jobTemplate` | Modèle du `Job` créé à chaque déclenchement. |
| `restartPolicy: OnFailure` | Redémarre le conteneur en cas d'échec. **Obligatoire** pour un Job (`Never` ou `OnFailure`). |
| `image: busybox:1.36` | Image légère qui contient la commande `date`. |
| `command: ["date"]` | Remplace l'entrypoint du conteneur : exécute simplement `date`. |

### Comprendre `*/5 * * * *`

La planification cron comporte 5 champs :

```text
* * * * *
│ │ │ │ │
│ │ │ │ └── jour de la semaine (0-7, 0 et 7 = dimanche)
│ │ │ └──── mois (1-12)
│ │ └────── jour du mois (1-31)
│ └──────── heure (0-23)
└────────── minute (0-59)
```

`*/5` dans le champ « minute » signifie : **toutes les 5 minutes**, c'est-à-dire à
`00`, `05`, `10`, `15`, `20`, … de chaque heure.

Quelques exemples :

| Schedule | Signification |
|---|---|
| `*/5 * * * *` | Toutes les 5 minutes |
| `0 * * * *` | Toutes les heures, à la minute 0 |
| `0 2 * * *` | Tous les jours à 02:00 |
| `0 2 * * 1` | Tous les lundis à 02:00 |

> **Astuce :** Le CronJob utilise le fuseau horaire du controller-manager Kubernetes (souvent **UTC** sur
> Minikube). Le champ `timeZone` (Kubernetes ≥ 1.27) permet de le préciser, par exemple
> `timeZone: "Europe/Paris"`.

---

## 3. Ce qui se passe concrètement

```text
CronJob "date"
   │  toutes les 5 minutes
   ▼
Job "date-<timestamp>"
   │
   ▼
Pod "date-<timestamp>-<suffixe>"
   │
   └── exécute `date` puis se termine
```

1. Toutes les 5 minutes, le controller du CronJob crée un objet **Job**.
2. Le Job crée un **Pod**.
3. Le Pod exécute `date` et affiche le résultat sur sa sortie standard.
4. Le Pod passe en `Completed`, et le Job est marqué comme terminé.
5. Le résultat n'apparaît **pas** dans le terminal : il faut consulter les **logs** du Pod/Job.

Les anciens Jobs sont conservés selon `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`,
puis nettoyés automatiquement.

---

## 4. Commandes utilisées

### Déployer le CronJob

```bash
kubectl apply -f TP/cronjob/cronjob.yml
```

Résultat :

```text
cronjob.batch/date created
```

### Vérifier le CronJob

```bash
kubectl get cronjob date
# ou
kubectl get cj
```

```text
NAME   SCHEDULE      TIMEZONE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE   CONTAINERS   IMAGES         SELECTOR
date   */5 * * * *   Europe/Paris   False     0        <none>          0s    date         busybox:1.36   <none>
```

- `SUSPEND: False` : le CronJob est actif.
- `ACTIVE: 0` : aucune exécution en cours.
- `LAST SCHEDULE: <none>` : pas encore déclenché (il attend la prochaine minute multiple de 5).

Pour voir la configuration complète :

```bash
kubectl describe cronjob date
```

### Tester immédiatement (sans attendre 5 minutes)

```bash
kubectl create job --from=cronjob/date date-manuel
kubectl wait --for=condition=complete job/date-manuel --timeout=120s
kubectl logs job/date-manuel
```

Résultat observé :

```text
job.batch/date-manuel created
job.batch/date-manuel condition met
Fri Nov 13 14:08:34 UTC 2026
```

Cela prouve que le modèle de Job du CronJob fonctionne. Le Job `date-manuel` est un test
manuel : il ne fait pas partie des exécutions planifiées.

### Vérifier les Jobs et Pods créés

```bash
kubectl get jobs
kubectl get pods
```

Un Pod terminé correctement affiche :

```text
NAME                         READY   STATUS      RESTARTS   AGE
date-<timestamp>-<suffixe>   0/1     Completed   0          1m
```

### Attendre la prochaine exécution planifiée

```bash
# Suivre la création des Jobs en direct
kubectl get jobs -w
```

Puis, dans un autre terminal :

```bash
kubectl get pods
kubectl logs job/<nom-du-job-créé>
```

Il faut attendre la prochaine minute multiple de 5 (par exemple 14:10, 14:15, etc.).

Résultat observé après la planification :

```text
NAME                 SCHEDULE      TIMEZONE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/date   */5 * * * *   Europe/Paris   False     0        35s             2m4s

NAME                      STATUS     COMPLETIONS   DURATION   AGE
job.batch/date-29909650   Complete   1/1           3s         35s

NAME                          READY   STATUS      RESTARTS   AGE
pod/date-29909650-xpk9m       0/1     Completed   0          35s
```

Logs du Job planifié :

```bash
kubectl logs job/date-29909650
```

```text
Fri Nov 13 14:10:00 UTC 2026
```

Le CronJob a donc bien créé un Job à **14:10:00**, soit exactement 5 minutes après le début de la
période. Le Pod s'est terminé en `Completed`.

> **Remarque :** ce log vient du premier essai, avant l'ajout de `timeZone`. Depuis, le
> controller planifie en heure de Paris, mais le `date` du conteneur continue d'afficher UTC :
> `timeZone` change l'heure de déclenchement, pas le fuseau horaire à l'intérieur du conteneur.

### Nettoyer le test manuel

```bash
kubectl delete job date-manuel
```

> Ne supprimez pas le CronJob si vous voulez qu'il continue à s'exécuter.

### Supprimer le CronJob

```bash
kubectl delete -f TP/cronjob/cronjob.yml
# ou
kubectl delete cronjob date
```

> **Attention :** Supprimer le CronJob ne supprime pas forcément les Jobs déjà créés. Pour tout nettoyer :
> ```bash
> kubectl delete jobs -l ...
> ```
> Le plus simple sur un TP est de supprimer les Jobs manuellement :
> ```bash
> kubectl get jobs
> kubectl delete job <nom-du-job>
> ```

---

## 5. Livrables

- [x] `TP/cronjob/cronjob.yml`
- [x] Liste des commandes utilisées (ce document)
- [x] Test immédiat avec `kubectl create job --from=cronjob/date date-manuel`
- [x] Exécution planifiée vérifiée (Job `date-29909650` créé à 14:10:00)
- [x] Résultat de la commande `date` visible dans les logs

---

## 6. Ce que j'ai retenu

Le CronJob en lui-même est très court à écrire : le plus long, c'est de comprendre où lire le
résultat. Le Pod se termine en `Completed`, donc `kubectl logs pod/...` ne suffit pas toujours :
il faut viser le Job (`kubectl logs job/date-...`) ou le Pod associé.

J'ai aussi ajouté `timeZone: "Europe/Paris"` parce que le controller tournait en UTC. Sans ça,
l'exécution de 14:10 UTC correspondait à 15:10 à Paris, ce qui n'était pas ce que j'attendais.
C'est un détail tout bête, mais qui change l'heure des backups en production.
