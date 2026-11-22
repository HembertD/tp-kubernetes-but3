# Rendu — TP Kubernetes

Ce dossier regroupe les comptes rendus et les captures d'écran du TP.

## Documents

- [Partie 1 — Namespaces et NGINX](partie1-reponses.md)
- [Partie 2 — MySQL, WordPress et NGINX](partie2-reponses.md)
- [Partie 3 — CronJob](partie3-cronjob-reponses.md)
- [Partie 3 — HPA (étapes 1 à 9)](partie3-hpa-reponses.md)
- [Partie 4 — QCM](partie4-reponses.md)
- [Partie 5 — Application production-ready](partie5-reponses.md)

## Captures d'écrans

Les captures que j'ai réalisées pour répondre aux questions sont dans [`img/`](img/) :

| Image | Utilisée dans |
|---|---|
| `img/partie1-namespaces-avant.png` | Partie 1 — namespaces avant création (namespaces système) |
| `img/liste_namespace_TP_partie1.png` | Partie 1 — namespaces après création |
| `img/partie1-nginx.png` | Partie 1 — page par défaut NGINX |
| `img/partie2-mysql-variables.png` | Partie 2 — variables d'environnement MySQL |
| `img/partie2-wordpress-variables.png` | Partie 2 — variables d'environnement WordPress |
| `img/partie2-wordpress-installation.png` | Partie 2 — page d'installation WordPress via NGINX |
| `img/partie3-hpa-application.png` | Partie 3 — application NGINX exposée par le Service |
| `img/partie3-hpa-scaling.png` | Partie 3 — HPA : 1 → 3 → 1 pendant le test de charge |
| `img/partie5-hello-kubernetes.png` | Partie 5 — page hello-kubernetes |
| `img/partie5-readiness-events.png` | Partie 5 — readiness et événements du pod |
| `img/partie5-rolling-update.png` | Partie 5 — RollingUpdate pod par pod |
| `img/partie5-prestop.png` | Partie 5 — arrêt gracieux avec preStop |
| `img/partie5-debug.png` | Partie 5 — conteneur éphémère de débogage |

## Checklist des captures demandées

| Partie | Étape | Capture |
|---|---|---|
| 1 | Namespaces avant création | `img/partie1-namespaces-avant.png` |
| 1 | Namespaces après création | `img/liste_namespace_TP_partie1.png` |
| 1 | Page NGINX | `img/partie1-nginx.png` |
| 2 | Variables MySQL | `img/partie2-mysql-variables.png` |
| 2 | Variables WordPress | `img/partie2-wordpress-variables.png` |
| 2 | Page WordPress via NGINX | `img/partie2-wordpress-installation.png` |
| 3 | Test de charge HPA | `img/partie3-hpa-scaling.png` |
| 3 | Application exposée (bonus) | `img/partie3-hpa-application.png` |
| 5 | Page hello-kubernetes | `img/partie5-hello-kubernetes.png` |
| 5 | Readiness et Events | `img/partie5-readiness-events.png` |
| 5 | RollingUpdate | `img/partie5-rolling-update.png` |
| 5 | Débogage `kubectl debug` | `img/partie5-debug.png` |
| 5 | Arrêt gracieux (bonus) | `img/partie5-prestop.png` |

## Fichiers YAML

Les manifestes Kubernetes livrables restent dans le dossier [`TP/`](../TP/) :

```text
TP/
├── front/
│   ├── namespace.yml
│   ├── nginx-cm.yml
│   ├── nginx-deployment.yml
│   └── nginx-service.yml
├── middle/
│   ├── namespace.yml
│   ├── mysql-configmap.yml
│   ├── mysql-secret.yml
│   ├── wordpress-deployment.yml
│   ├── wordpress-pv.yml
│   ├── wordpress-pvc.yml
│   └── wordpress-service.yml
├── database/
│   ├── namespace.yml
│   ├── mysql-configmap.yml
│   ├── mysql-secret.yml
│   ├── mysql-service.yml
│   └── mysql-statefulset.yml
├── cronjob/
│   └── cronjob.yml
├── hpa/
│   ├── application.yml
│   └── hpa-application.yml
└── prod-app/
    ├── namespace.yml
    ├── app-deployment.yml
    └── app-service.yml
```

> Les commandes `kubectl apply -f TP/...` présentes dans les comptes rendus sont à exécuter
> **depuis la racine du dépôt**, pas depuis ce dossier.
