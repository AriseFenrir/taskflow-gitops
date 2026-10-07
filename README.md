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
![alt text](images-matin/image.png)

On ne peux pas push :
![alt text](images-matin/image-1.png)

### Etape 2
ajout du pseudo dans application.yaml
![alt text](images-matin/image-2.png)

lancement du script d'installation
![alt text](images-matin/image-3.png)

on apply le manifeste Argo CD
![alt text](images-matin/image-4.png)

![alt text](images-matin/image-17.png)

### Etape 3
On vérifie que l'application est bien synchronisée dans Argo CD (sync status et health status).
![alt text](images-matin/image-5.png)

on lance le script d'observation pour vérifier la répartition du trafic
![alt text](images-matin/image-6.png)

### Etape 4
Création de la première Pull Request pour déployer la nouvelle version de TaskFlow.
![alt text](images-matin/image-7.png)

on a appliquer la pr et l'application est synchronisée dans Argo CD.(pr applique a 11h52) au bout de 1 minute environ car dans le screen le changment a eu lieu a 11h53
![alt text](images-matin/image-8.png)

### Etape 5
on a mis le nombre de replicas a 1
![alt text](images-matin/image-10.png)

mais on remaque que tout de suite argo cd repasse a 4 replicas, car il aligne l'état du cluster sur l'état voulu défini dans Git.
![alt text](images-matin/image-9.png)

on test de changer l'image manuellement et dans l'observation on remarque qu'elle repasse tout de suite a 2.0.0
![alt text](images-matin/image-11.png)

![alt text](images-matin/image-12.png)

### Etape 6
Pour le revert on a fait une autre pr pour ajouter les info du read me ce qui bloque le revert 
![alt text](images-matin/image-13.png)

afin de faire le revert on a donc fais une branche dans laquelle on a changer l'image et qu'on merge dans la main 

pr pour le revert
![alt text](images-matin/image-14.png)

a 12h22 on a merge la PR pour le revert et l'application est revenue à l'état précédent.
![alt text](images-matin/image-15.png)
et a 12h24 on remarque qu'on est bien repasser en 1.0.0
![alt text](images-matin/image-16.png)

## Bonus : suppression de service.yaml (Pruning)

Avant la suppression, le Service `taskflow` est bien présent dans l'arborescence d'Argo CD :
![alt text](images-matin/image-18.png)

1. Création de la branche `feat/bonus-prune-service` et suppression du fichier `apps/taskflow/service.yaml`.
2. Ouverture et fusion de la Pull Request #8 sur `main`.
3. Grâce à `prune: true` dans `argocd/application.yaml`, Argo CD a automatiquement supprimé (pruné) la ressource `Service` du cluster :
![alt text](images-matin/image-19-clean.png)

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

### Partie A : Blue Green Deployment

### Etape 1

On commence par recuperer les fichier bluegreen dans le dépôt Git. quon va merge dans la main avant de faire la partie 1.1.0

![alt text](images-aprem/image.png)

Une fois merge on va verifie sur argo CD que les changements ont bien été pris en compte.
![alt text](images-aprem/image-1.png)
Rollout taskflow créé avec stratégie BlueGreen, 4 pods en revision:1
Status : Healthy, image 1.0.0 marquée stable, active
2 Services créés : taskflow (production) et taskflow-preview
observe.sh : 40/40 requêtes → version=1.0.0 http=200*

### Etape 2
Maintenant, on peut passer à la mise à jour de l'image vers la version 1.1.0 et observer le comportement du déploiement BlueGreen.

On met à jour le fichier `apps/taskflow/rollout.yaml` pour changer l'image de la version 1.0.0 à 1.1.0, puis on commit et push les changements vers le dépôt Git.
![alt text](images-aprem/image-2.png)

Une fois merge on peut abserver le rollout sur Argo CD et vérifier que la nouvelle version 1.1.0 est bien déployée.
![alt text](images-aprem/image-3.png)
on peut voir les 4 pod en version 1.1.0.

Il y a donc 8 pod au total 4 en version 1.0.0 et 4 en version 1.1.0.

on regarde egalement le script observe dans lequel il reste afficher la version 1.0.0 car cela reste la version en prod 
![alt text](images-aprem/image-4.png)

### Etape 3

On lance la promotion :
![alt text](images-aprem/image-5.png)

et on observe que les pod 1.0.0 ne sont plus utiliser et que les pod actif sont desormais ceux en 1.1.0
![alt text](images-aprem/image-6.png)

et dans le script observe.sh on voit la version 1.1.0 
![alt text](images-aprem/image-7.png)

et apres 30s on peut voir que les pods de l'ancienne version 1.0.0 ont été supprimés et que seuls les pods en version 1.1.0 sont actifs.
![alt text](images-aprem/image-8.png)

### Partie B : Canary Deployment

### Etape 1

On commence par recuperer les fichier canary dans le dépôt Git. quon va merge dans la main 

![alt text](images-aprem/image-9.png)

Une fois merge on va verifie sur argo CD que les changements ont bien été pris en compte.

![alt text](images-aprem/image-10.png)

Rollout taskflow créé avec stratégie Canary, 4 pods en revision:1
Status : Healthy, image 1.1.0 marquée stable, active

![alt text](images-aprem/image-11.png)

observe.sh : 40/40 requêtes → version=1.1.0 http=200

### Etape 2

On met à jour le fichier `apps/taskflow/rollout.yaml` pour changer l'image de la version 1.1.0 à 2.0.0, puis on commit et push les changements vers le dépôt Git.

![alt text](images-aprem/image-12.png)

On peut alors observer que le rollout a commencer et que la nouvelle version est progressivement déployée.

il commence a 25% soit 1 pod sur 4 en version 2.0.0.
![alt text](images-aprem/image-13.png)

et sur observe.sh on remarque environ 25% des requete en version 2.0.0
![alt text](images-aprem/image-14.png)

on va ensuite passer à 50% soit 2 pods sur 4 en version 2.0.0.
![alt text](images-aprem/image-15.png)

![alt text](images-aprem/image-16.png)

on passe ensuite a 75% soit 3 pods sur 4 en version 2.0.0.

![alt text](images-aprem/image-17.png)

et enfin a 100% soit 4 pods sur 4 en version 2.0.0.

![alt text](images-aprem/image-18.png)

et sur observe .sh on remarque que toutes les requêtes sont maintenant en version 2.0.0.
![alt text](images-aprem/image-19.png)

### Etape 3

On va maintenant passez en version 2.1.0. afin de faire l'observation des code http.

![alt text](images-aprem/image-20.png)

On va alors pouvoir observer les pods en version 2.1.0 et les codes HTTP associés.

![alt text](images-aprem/image-21.png)

et avec observe.sh on peut voir la version 2.1.0 et les codes HTTP associés on des erreur 500. (les 200 de 2.1.0 baisse et les erreur 500 augmentent)
![alt text](images-aprem/image-22.png)

On remarque alors que la nouvelle version 2.1.0 provoque des erreurs 500, ce qui indique un problème avec cette version.

On va lancer un abort du rollout pour revenir à la version stable précédente.
![alt text](images-aprem/image-23.png)

et on peut abserver que le pod 2.1.0 a ete kill et que le pod a ete recreer en version 2.0.0
![alt text](images-aprem/image-24.png)

et dans observe.sh on ne pert plus de requete
![alt text](images-aprem/image-25.png)

## Blue-Green ou Canary pour TaskFlow ? 

Pour TaskFlow, la stratégie **Canary** est la plus adaptée, pour deux raisons principales :

### Coût
- En Blue-Green, le déploiement nécessite **8 pods** (le double de la production) pendant toute la phase de validation. Cela double temporairement la consommation de ressources (CPU, mémoire).
- En Canary, le nombre de pods reste **constant à 4**. À 25%, un pod de l'ancienne version est remplacé par un pod de la nouvelle. Il n'y a aucun surcoût en ressources.
- Pour une application comme TaskFlow qui tourne avec 4 réplicas, le Canary est nettement plus économique.

### Risque
- En Blue-Green, aucun utilisateur ne voit la nouvelle version avant la promotion. C'est plus sûr **en théorie**, mais si un bug passe les tests sur le service preview, il touche **100% des utilisateurs d'un coup** à la promotion.
- En Canary, on expose d'abord **25% du trafic réel**. Comme on l'a vu avec la version 2.1.0, les erreurs 500 ont été détectées immédiatement grâce à `observe.sh`. L'abort a permis de revenir à la version stable avant que la majorité des utilisateurs soient impactés.
- Le Canary offre donc une **détection en conditions réelles** avec un **blast radius limité**.

### Conclusion
Le Canary est le meilleur compromis pour TaskFlow : il ne coûte rien de plus en ressources et permet de détecter les problèmes en production réelle tout en limitant l'impact sur les utilisateurs. Le Blue-Green reste pertinent pour des applications critiques où **aucun** utilisateur ne doit être exposé à une version non validée (ex: paiement, données sensibles).
