# taskflow-gitops — dépôt GitOps du cours CI/CD M2

Ce dépôt décrit **l'état voulu** de l'application TaskFlow dans Kubernetes.
Argo CD le surveille et aligne le cluster dessus : pour changer la production,
on ne tape pas de commande, on fait une **Pull Request**.

## Équipe

- Dubois Thomas
- Jennifer Vernet

## Installation

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

## Structure du dépôt

| Chemin | Rôle |
| --- | --- |
| `apps/taskflow/` | Manifests surveillés par Argo CD |
| `argocd/application.yaml` | Déclaration de l'application dans Argo CD |
| `exemples/bluegreen/` | Manifests pour le déploiement Blue-Green |
| `exemples/canary/` | Manifests pour le déploiement Canary |
| `exemples/robustesse/` | Canary avec test de charge k6 automatique, modèle de postmortem |
| `scripts/install.sh` | Installation de l'environnement |
| `scripts/argocd-ui.sh` | Ouvre l'interface d'Argo CD |
| `scripts/observe.sh` | Montre quelle version répond et le code HTTP |
| `scripts/charge.sh` | Lance le test de charge k6 contre un service |

## Images disponibles

`ghcr.io/9m7fjfpv9k-cyber/taskflow` en versions `1.0.0`, `1.1.0`, `2.0.0`, `2.1.0` et `2.2.0`.

## Journaux de lab

| Jour | Contenu | Lien |
| --- | --- | --- |
| **J2** | GitOps (Argo CD, self-heal, revert), Blue-Green, Canary manuel | [docs/journal-j2.md](docs/journal-j2.md) |
| **J3 matin** | Robustesse (analyse automatique k6, incident 2.1.0, postmortem) | [docs/journal-j3.md](docs/journal-j3.md) |
| **J3 après-midi** | Mini-PSSI, quality gates (conftest, Trivy, checks CI obligatoires) | [docs/journal-j3.md](docs/journal-j3.md#partie-c--la-mini-pssi-en-quality-gates) |
