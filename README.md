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
- Jallow Abdoulie (`jabdoulie`)
- Hannan Hassif (`123Hassif`)

## Journal des déploiements

Cluster `kind-cicd`. Argo CD surveille `https://github.com/jabdoulie/taskflow-gitops.git`, branche `main`, avec `automated.prune` et `automated.selfHeal`.

### Réponses

**Pull.** Argo CD va chercher l'état dans Git. Après chaque merge sur `main`, l'interface passe à **Synced** toute seule (`Auto sync is enabled`). Aucun `kubectl apply` des manifests n'est refait : le seul `kubectl apply` du journal sert à déclarer l'Application.

**Qui a corrigé quoi.**

| Changement | Auteur | Relecture et merge | Effet sur le cluster |
| --- | --- | --- | --- |
| PR #1, fork et binôme (`993090b`) | jabdoulie | 123Hassif | Application Healthy sur le fork |
| PR #2, image `2.0.0` (`7bd9009`) | 123Hassif | jabdoulie | Déploiement en `2.0.0` |
| Dérive `scale` à 1 et image `1.1.0` | jabdoulie, en direct sur le cluster | Argo CD (`selfHeal`) | Retour à 4 replicas et à l'image de Git |
| PR #3, revert de la #2 (`7028150`) | jabdoulie | 123Hassif | Retour de l'image à `1.0.0` |
| PR #4, suppression de `service.yaml` (`4cd5f52`) | jabdoulie | 123Hassif | Le Service disparaît (`prune`) |

**Pourquoi `git revert`.** Le bouton Revert ajoute un commit qui annule la PR #2, puis ce commit passe par une PR. L'historique de `main` reste en place, le ruleset n'est pas contourné, et Argo CD déploie ce nouveau commit (`7028150`) : l'image redevient `1.0.0`.

### 1. Premier déploiement en `1.0.0`

Contexte `kind-cicd`, Application créée, puis `./scripts/observe.sh` : les 40 requêtes répondent `version=1.0.0` en HTTP 200.

![Contexte kind-cicd, apply de l'Application, observe en 1.0.0](docs/journal/01-apply-observe-1.0.0.png)

L'application est **Healthy** et **Synced** sur `main`. Le Service, le Deployment et les 4 pods sont là.

![Argo CD Healthy et Synced après la connexion au fork](docs/journal/02-synced-pr1.png)

Le commit déployé ensuite est le merge de la PR #1 (`993090b`), le 7 octobre 2026 à 10:49. Hannan (`123Hassif`) a mergé le commit qui pointe Argo CD vers le fork.

![Détail du commit 993090b, merge de la PR #1](docs/journal/03-commit-pr1.png)

### 2. PR image `2.0.0`

PR #2, branche `update-image-version-2.0.0` : Hannan passe l'image de `1.0.0` à `2.0.0`. La PR attend une approbation.

![PR #2, diff de l'image vers 2.0.0](docs/journal/04-pr2-image-2.0.0.png)

Après le merge par jabdoulie (`7bd9009`), Argo CD se synchronise seul. Le Deployment passe en révision 2.

![Sync OK sur 7bd9009, merge de la PR #2](docs/journal/05-synced-pr2.png)

Le manifeste vivant du pod confirme l'image `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.0.0` (ReplicaSet `745bdbf6f8`).

![Manifeste vivant, image 2.0.0](docs/journal/06-live-manifest-2.0.0.png)

### 3. Dérive, puis self-heal

Commandes lancées sur le cluster, en dehors de Git : 1 replica, image `1.1.0`.

![kubectl scale à 1 et set image 1.1.0](docs/journal/07-derive-kubectl.png)

Argo CD corrige. Quatre pods repassent **Running** sur le ReplicaSet de Git (`745bdbf6f8`, image `2.0.0`). Les pods de la dérive passent **Terminating**. Le Service est encore présent.

![Self-heal : 4 pods sur le ReplicaSet de Git, les autres s'arrêtent](docs/journal/08-selfheal.png)

### 4. Revert vers `1.0.0`

PR #3, bouton Revert de la PR #2. jabdoulie ouvre le revert, 123Hassif l'approuve et le merge (`7028150`).

![PR #3 mergée, revert de la mise à jour 2.0.0](docs/journal/09-pr3-revert.png)

À 11:02 les pods `1.0.0` (`76547969d7`) avaient été supprimés pour laisser place à `2.0.0`. À 11:09 le revert les recrée.

![Événements : suppression des pods 1.0.0 à 11:02, recréation à 11:09](docs/journal/10-events-retour-1.0.0.png)

**Synced** sur `7028150`. Le ReplicaSet `76547969d7` (rév. 5) porte de nouveau les 4 pods. Le ReplicaSet `2.0.0` (`745bdbf6f8`) ne reçoit plus de trafic.

![Sync OK sur 7028150, retour des pods 1.0.0](docs/journal/11-synced-revert.png)

### 5. Bonus : prune du Service

PR #4, suppression de `apps/taskflow/service.yaml`. 123Hassif approuve et merge (`4cd5f52`).

![PR #4 mergée, service.yaml retiré](docs/journal/12-pr4-prune.png)

Quelques secondes plus tard, Argo CD est **Synced** sur `4cd5f52`. Le nœud Service a disparu de l'arbre. Le Deployment et les pods restent.

![Sync OK sur 4cd5f52, le Service n'est plus dans l'arbre](docs/journal/13-synced-prune.png)
