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

Elias 

## Captures

### Dernière PR et déploiement

![Dernière PR](docu/image-copy.png)

La capture montre la date du dernier merge de la PR, et le déploiement a été appliqué par Argo CD en moins d’une minute.

### Dernière synchronisation Argo CD

![Dernière synchronisation](docu/image-copy-2.png)

La capture montre la date et les minutes de la dernière synchronisation Argo CD, avec le projet `taskflow` en état `Synced` et `Healthy`.

### Vérification du namespace `taskflow`

![Namespace taskflow](docu/imagetest.png)

Cette capture illustre le résultat du script `./scripts/observe.sh` : le namespace `taskflow` existe bien et le service répond correctement.

### Dérive manuelle corrigée immédiatement

![Dérive manuelle](docu/image-copy-3.png)

![Restauration Argo CD](docu/image-copy-4.png)

Lorsqu’on exécute une commande telle que `kubectl scale deployment/taskflow --replicas=0`, le cluster diverge de l’état décrit dans Git. Argo CD le détecte immédiatement et réapplique automatiquement la configuration voulue, de sorte que la version de base est restaurée quasi instantanément. C’est le principe du GitOps : Git reste la source de vérité, et Argo CD corrige automatiquement les écarts du cluster.

### Temps de synchronisation Argo CD

Le déploiement n’est pas instantané à l’instant précis du merge : Argo CD applique les changements au prochain cycle de reconciliation. Dans ce projet, la fréquence a été configurée à 60 secondes via la configuration de Argo CD, ce qui explique pourquoi le passage d’une version à l’autre a pris environ une minute. La synchronisation est donc dépendante du polling Argo CD et du temps de comparaison entre l’état Git et l’état réel du cluster.

### Revert de la PR et retour à la version précédente

![Revert de la PR](docu/image-copy-5.png)

![Retour à la version précédente](docu/image-copy-6.png)

Après avoir effectué le revert de la PR, le déploiement est immédiatement ramené à sa version précédente. La transition de `2.0.0` vers `1.0.0` est ainsi rétablie presque instantanément par Argo CD, ce qui confirme que la branche Git reste la source de vérité et que le cluster suit automatiquement l’état voulu.

### Prune après suppression du fichier `service.yaml`

![Suppression du service par PR](docu/image-copy-7.png)

![Service supprimé dans le cluster](docu/image-copy-8.png)

Après avoir supprimé le fichier `service.yaml` dans la PR, Argo CD a appliqué automatiquement le prune. Le `Service` n’apparaît plus dans le namespace `taskflow`, ce qui montre que les ressources absentes du dépôt Git sont supprimées du cluster par Argo CD. C’est le comportement attendu du mode `prune: true` dans le GitOps.
