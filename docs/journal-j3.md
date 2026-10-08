# Journal Jour 3 — Robustesse, analyse automatique et PSSI

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

---

## Partie C : La mini-PSSI en quality gates

### Etape 1 

On commence par lancer le conftest:

![alt text](image.png)

Actuellement tout les tests configurer passent.

### Etape 2 : Écriture des règles R3 et R4

Ecriture de R3 :
![alt text](image-1.png)

Ecriture de R4 :

![alt text](image-2.png)

Après relance de conftest, **R4 échoue** car le rollout n'a pas de `securityContext.runAsNonRoot`.

<!-- Capture de l'échec R4 -->
<!-- ![alt text](../images-j3/image-XX.png) -->

### Etape 3 : Correction du rollout pour R4

On ajoute `securityContext: runAsNonRoot: true` dans `apps/taskflow/rollout.yaml` au niveau du pod.

Relance de conftest : toutes les règles R1 à R4 passent.

<!-- Capture de conftest OK -->
<!-- ![alt text](../images-j3/image-XX.png) -->

### Etape 4 : Mise en place du workflow CI

On copie le workflow GitHub Actions :
```bash
cp exemples/ci/pssi-github.yml .github/workflows/pssi.yml
```

PR `feat/pssi-quality-gates` avec les 3 fichiers modifiés (rego, rollout, workflow), merge dans main.

<!-- Capture de la PR et des checks CI -->
<!-- ![alt text](../images-j3/image-XX.png) -->

### Etape 5 : Ruleset — checks obligatoires

Dans GitHub → Settings → Rules, on ajoute les deux status checks obligatoires :
- **PSSI manifests (conftest)**
- **PSSI images (Trivy)**

<!-- Capture de la ruleset -->
<!-- ![alt text](../images-j3/image-XX.png) -->

### Etape 6 : PR non conforme (test du blocage)

On crée une PR avec une image non conforme (`nginx:latest`) pour vérifier que les quality gates bloquent bien le merge.

Les checks échouent :
- R1 : tag `latest` interdit
- R2 : image `nginx` hors du registre autorisé

<!-- Capture de la PR bloquée -->
<!-- ![alt text](../images-j3/image-XX.png) -->

### Etape 7 : Trivy — gestion des vulnérabilités

<!-- Si Trivy échoue : capture + correction ou exception dans .trivyignore -->
<!-- ![alt text](../images-j3/image-XX.png) -->

### Tableau récapitulatif : Règle → Contrôle → Outil → Preuve

| Règle | Exigence | Outil | Résultat |
| --- | --- | --- | --- |
| **PSSI-R1** | Tag explicite, jamais `latest` | conftest | PR `nginx:latest` bloquée |
| **PSSI-R2** | Registre `ghcr.io/9m7fjfpv9k-cyber/` uniquement | conftest | PR `nginx` bloquée |
| **PSSI-R3** | Limite de mémoire sur chaque conteneur | conftest | Validé (256Mi configuré) |
| **PSSI-R4** | `runAsNonRoot: true` sur les pods | conftest | Corrigé dans le rollout |
| **PSSI-R5** | Aucune CVE HIGH/CRITICAL corrigeable | Trivy | Scanné en CI |
