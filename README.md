# taskflow-gitops — dépôt GitOps du cours CI/CD M2

Ce dépôt décrit **l'état voulu** de l'application TaskFlow dans Kubernetes.
Argo CD le surveille et aligne le cluster dessus : pour changer la production,
on ne tape pas de commande, on fait une **Pull Request**.

## Installation (à faire chez vous, avant le cours)

Prérequis : Docker Desktop démarré, 8 Go de RAM, 10 Go de disque libre.
Sous Windows : WSL2 (Ubuntu) + intégration WSL de Docker Desktop, et toutes les commandes dans WSL.

```bash
git clone https://github.com/9m7fjfpv9k-cyber/taskflow-gitops.git
cd taskflow-gitops
./scripts/install.sh
```

Le script crée un cluster local `kind`, installe Argo CD et Argo Rollouts,
puis télécharge les images des labs. Comptez 5 à 15 minutes.
Il peut être relancé sans risque.

## Structure

| Chemin | Rôle |
| --- | --- |
| `apps/taskflow/` | Les manifests surveillés par Argo CD |
| `argocd/application.yaml` | Déclare l'application dans Argo CD |
| `exemples/bluegreen/` | Manifests pour le déploiement Blue-Green |
| `exemples/canary/` | Manifests pour le déploiement Canary |
| `scripts/install.sh` | Installation de l'environnement |
| `scripts/argocd-ui.sh` | Ouvre l'interface d'Argo CD |
| `scripts/observe.sh` | Montre quelle version répond, et avec quel code HTTP |

## Images disponibles

`ghcr.io/9m7fjfpv9k-cyber/taskflow` en versions `1.0.0`, `1.1.0`, `2.0.0` et `2.1.0`.

## Équipe

<!-- Noms du binôme -->
- Dubois Thomas
- Jennifer Vernet

## LAB MATIN
### Etape 1 
ruleset creer sur main
![alt text](images/image.png)

On ne peux pas push :
![alt text](images/image-1.png)

### Etape 2
ajout du pseudo dans application.yaml
![alt text](images/image-2.png)

lancement du script d'installation
![alt text](images/image-3.png)

on apply le manifeste Argo CD
![alt text](images/image-4.png)

![alt text](images/image-17.png)

### Etape 3
On vérifie que l'application est bien synchronisée dans Argo CD (sync status et health status).
![alt text](images/image-5.png)

on lance le script d'observation pour vérifier la répartition du trafic
![alt text](images/image-6.png)

### Etape 4
Création de la première Pull Request pour déployer la nouvelle version de TaskFlow.
![alt text](images/image-7.png)

on a appliquer la pr et l'application est synchronisée dans Argo CD.(pr applique a 11h52) au bout de 1 minute environ car dans le screen le changment a eu lieu a 11h53
![alt text](images/image-8.png)

### Etape 5
on a mis le nombre de replicas a 1
![alt text](images/image-10.png)

mais on remaque que tout de suite argo cd repasse a 4 replicas, car il aligne l'état du cluster sur l'état voulu défini dans Git.
![alt text](images/image-9.png)

on test de changer l'image manuellement et dans l'observation on remarque qu'elle repasse tout de suite a 2.0.0
![alt text](images/image-11.png)

![alt text](images/image-12.png)

### Etape 6
Pour le revert on a fait une autre pr pour ajouter les info du read me ce qui bloque le revert 
![alt text](images/image-13.png)

afin de faire le revert on a donc fais une branche dans laquelle on a changer l'image et qu'on merge dans la main 

pr pour le revert
![alt text](images/image-14.png)

a 12h22 on a merge la PR pour le revert et l'application est revenue à l'état précédent.
![alt text](images/image-15.png)
et a 12h24 on remarque qu'on est bien repasser en 1.0.0
![alt text](images/image-16.png)

## Bonus : suppression de service.yaml (Pruning)

Avant la suppression, le Service `taskflow` est bien présent dans l'arborescence d'Argo CD :
![alt text](images/image-18.png)

1. Création de la branche `feat/bonus-prune-service` et suppression du fichier `apps/taskflow/service.yaml`.
2. Ouverture et fusion de la Pull Request #8 sur `main`.
3. Grâce à `prune: true` dans `argocd/application.yaml`, Argo CD a automatiquement supprimé (pruné) la ressource `Service` du cluster :
![alt text](images/image-19.png)

Vérification via le terminal :
```bash
$ kubectl get svc -n taskflow
No resources found in taskflow namespace.
```

## Questions du livrable

**Push ou Pull ?**
Argo CD fonctionne en mode Pull. C'est lui qui interroge le dépôt Git toutes les 60 secondes pour vérifier s'il y a des changements. Personne ne "pousse" vers le cluster — c'est Argo CD qui tire l'état depuis Git et l'applique.

**Qui a corrigé quoi ?**
Lors de la dérive, c'est Argo CD qui a corrigé automatiquement les modifications manuelles (kubectl scale, kubectl set image). Grâce à `selfHeal: true`, il a détecté que l'état du cluster ne correspondait plus à l'état décrit dans Git, et il a resynchronisé le cluster sur le dépôt Git (source de vérité).

**Pourquoi git revert ?**
Parce que dans une approche GitOps, le dépôt Git est la seule source de vérité. On ne fait jamais de modification directement sur le cluster. Pour revenir en arrière (ex: de 2.0.0 à 1.0.0), on ne fait pas `kubectl set image` — on fait un revert de la PR sur GitHub, ce qui recrée l'ancien état dans Git. Argo CD détecte le changement et redéploie automatiquement. Tout passe par Git = traçabilité complète, audit, historique, et review par PR.

## LAB APRÈS-MIDI