# Postmortem — Incident version 2.1.0 : erreurs 500 en production

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 2026-10-08, vers 10h40 |
| Version en cause | `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0` |
| PR à l'origine | PR feat/robustesse-2.1.0 (bump image 2.0.0 → 2.1.0) |
| Durée d'exposition | ~15 minutes (le temps de détecter, investiguer et revert) |
| Part du trafic touché | 100% — la 2.1.0 a été promue à l'ensemble des pods (voir section "Premier déploiement") |
| Détecté par | `observe.sh` (humain) lors du premier déploiement, puis test de charge k6 (automatique) lors du second déploiement |
| Résolu par | Revert via PR Git (image remise à 2.0.0), puis correction avec la version 2.2.0 |

## Chronologie

| Heure | Événement |
| --- | --- |
| ~10h40 | **Premier déploiement 2.1.0** — PR mergée, rollout canary lancé |
| ~10h40 | Canary à 25%, l'AnalysisRun k6 se lance |
| ~10h41 | Le Job k6 **échoue** (30% d'erreurs 500, seuils dépassés), mais l'AnalysisRun rapporte **Successful** |
| ~10h42 | Le rollout continue malgré l'échec réel du test → la 2.1.0 est promue à 100% |
| ~10h45 | `observe.sh` révèle le problème : 25 requêtes HTTP 200, 15 requêtes HTTP 500 (37.5% d'erreurs) |
| ~10h50 | Investigation : le Job k6 a bien détecté les erreurs (`thresholds crossed`), mais le code de sortie n'a pas déclenché l'échec de l'AnalysisRun car `abortOnFail` n'était pas configuré |
| ~11h00 | Revert PR : image remise à 2.0.0, ajout de `abortOnFail: true` dans les seuils k6 |
| ~11h05 | Problème de sync : le ConfigMap `k6-robustesse` n'est pas mis à jour par Argo CD — application manuelle nécessaire via `kubectl apply` |
| ~11h10 | **Second déploiement 2.1.0** — cette fois l'AnalysisRun détecte correctement les erreurs, le Job k6 fail, et le rollout est **automatiquement aborté** |
| ~11h15 | Revert PR 2.1.0 → retour stable en 2.0.0 |
| ~11h20 | **Déploiement 2.2.0** — le test k6 passe, le canary progresse 25% → 50% → 75% → 100% avec succès |

## Composant défaillant et cause racine

- **Quel composant a échoué ?**
  L'image `taskflow:2.1.0` provoque des erreurs HTTP 500 sur l'endpoint `/tasks`. Environ 30% des requêtes échouent. L'endpoint `/health` continue de répondre correctement (HTTP 200).

  Preuve — logs du Job k6 :
  ```
  checks_succeeded...: 70.00% 210 out of 300
  checks_failed......: 30.00% 90 out of 300
  http_req_failed....: 30.00% 90 out of 300
  level=error msg="thresholds on metrics 'http_req_duration, http_req_failed' have been crossed"
  ```

- **Pourquoi les probes Kubernetes ne l'ont-elles pas vu ?**
  Les readiness et liveness probes sont configurées sur `/health`, un endpoint léger qui ne teste pas la logique métier. La version 2.1.0 passe le health check mais échoue sur `/tasks`. Les probes ne vérifient que la disponibilité du processus, pas le bon fonctionnement de l'application.

- **Pourquoi l'analyse automatique n'a pas fonctionné au premier déploiement ?**
  Le test k6 a bien détecté les erreurs et affiché `thresholds crossed` dans ses logs. Cependant, sans l'option `abortOnFail: true`, k6 terminait son exécution normalement (code de sortie ambigu). Le Job Kubernetes n'a pas propagé l'échec correctement à l'AnalysisRun d'Argo Rollouts, qui a donc rapporté "Successful".

- **Cause racine :**
  Double cause — (1) un bug applicatif dans la version 2.1.0 provoquant des erreurs 500 sur `/tasks`, et (2) une configuration insuffisante du test k6 : l'absence de `abortOnFail: true` dans les thresholds empêchait la propagation de l'échec vers Argo Rollouts.

## Ce qui a bien fonctionné

- **Le test k6 a détecté les erreurs** dès le premier déploiement (30% d'erreurs, seuils dépassés), même si le résultat n'a pas été correctement propagé.
- **Le mécanisme de revert par PR** a permis de revenir rapidement à la version stable 2.0.0.
- **Après correction de la configuration** (`abortOnFail: true`), le second déploiement de la 2.1.0 a été automatiquement aborté comme attendu.
- **Le déploiement de la 2.2.0** a validé que le pipeline fonctionne correctement de bout en bout : test k6 réussi → canary progressif → promotion à 100%.
- **L'approche GitOps** a garanti la traçabilité complète : chaque changement est une PR, chaque revert est documenté dans l'historique Git.
