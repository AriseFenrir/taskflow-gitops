# Journal Jour 3 — Robustesse et analyse automatique

## Partie A : L'étalon et l'analyse

### Etape 1 : Sync du fork et baseline

On fait la pr pour sync avec le fork 
![alt text](../images-j3/image.png)

On confirme que le nouveau setup est mis en place sur le cluster
![alt text](../images-j3/image-1.png)

### Etape 2 : Test de charge baseline (étalon)

On a lancer le script charge.sh pour effectuer le test de charge baseline.

![alt text](../images-j3/image-2.png)

Dans le resultat on peut observer : 
0.00% d'erreur, (seuil < 2%)
p95 = 10.35ms (seuil < 250ms)
730 requêtes, 100% en statut 200

### Etape 3 : PR feat/analyse-auto

les fichier ont tous été mis en place au moment ou on a sync ce qui fait que la PR n'a pas nécessité de modifications supplémentaires.

### Etape 4 : Vérification des ressources

<!-- Résultats de :
  kubectl get rollout -n taskflow
  kubectl get deployment -n taskflow (aucun)
  kubectl get analysistemplate -n taskflow
  kubectl get configmap k6-robustesse -n taskflow
  kubectl get svc -n taskflow
-->

On verifie que toute les ressources sont pretes et configuré
![alt text](../images-j3/image-3.png)

---

## Partie B : L'incident

### Etape 1 : Déploiement de la 2.1.0 (image défectueuse)

<!-- PR : image 2.1.0
  kubectl argo rollouts get rollout taskflow -n taskflow --watch
  NE RIEN TOUCHER — observer l'analyse automatique
-->

on fait la pr pour déployer l'image 2.1.0

![alt text](../images-j3/image-4.png)

le test avec k6 se lance a 25%
![alt text](../images-j3/image-7.png)

et on remarque apres le message d'erreur
![alt text](../images-j3/image-8.png)

### Etape 2 : Preuves de l'échec automatique

on peut check le fait que l'analysis run a bine echouer (ici c'est celle de 11m a prendre en compte)
![alt text](../images-j3/image-9.png)

on peut observer les log du job k6 dans lequel l'échec s'est produit.
![alt text](../images-j3/image-10.png)

on peut egalemnt check les log du analysis run 
![alt text](../images-j3/image-11.png)

Enfin on peut observer le statut du rollout qui est passé en Degraded.
![alt text](../images-j3/image-12.png)

### Etape 3 : Revert de la PR 2.1.0

On fait la PR pour revert la version 2.1.0 et revenir à la version précédente.
![alt text](../images-j3/image-13.png)

### Etape 4 : Déploiement de la 2.2.0 (image corrigée)

<!-- PR : image 2.2.0
  kubectl argo rollouts get rollout taskflow -n taskflow --watch
  L'analyse k6 passe → canary continue : 25% → 50% → 75% → 100%
-->
on fait la pr pour déployer l'image 2.2.0
![alt text](../images-j3/image-4.png)

le test avec k6 se lance a 25%
![alt text](../images-j3/image-14.png)

et on peut observer que le test a réussi et que le rollout continue.
![alt text](../images-j3/image-15.png)

le rollout fini avec succès.
![alt text](../images-j3/image-16.png)

### Etape 5 : Postmortem

Voir [postmortem-2.1.0.md](postmortem-2.1.0.md)

---

## Canary manuel vs Canary automatique

| Critère | Canary manuel (J2) | Canary avec analyse auto (J3) |
| --- | --- | --- |
| **Détection** | Humain via `observe.sh` | Job k6 automatique |
| **Décision d'abort** | `kubectl argo rollouts abort` manuel | Abort automatique par Argo Rollouts |
| **Temps de réaction** | Dépend de la vigilance de l'opérateur | ~30 secondes (durée du test k6) |
| **Intervention humaine** | Obligatoire | Aucune |
| **Risque d'oubli** | Élevé (nuit, week-end, distraction) | Nul |
