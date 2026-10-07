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

# LAB Blue-Green Canary

## Blue Green Part


### Changement de version dans le code

![Changement de version dans le code](docu2/image1.png)

Le manifeste a été modifié pour faire passer l’image de la version actuelle vers une version cible, en gardant le principe GitOps : le dépôt reste la source de vérité.

### Présence du système Green et du système Blue

![Système Green et Blue](docu2/image2.png)

Dans une stratégie Blue-Green, le système actif et le système de prévisualisation coexistent pendant la validation. Le trafic reste sur le service actif jusqu’à la promotion.

J'ai oublié de modif le observe.sh histoire de voir tous les pods mais en soit la vue IHM revient au même.

### Pré-promotion : sortie du script `observe.sh`

![Pré-promotion observe](docu2/image5.png)

Avant la promotion, le script `observe.sh` montre encore la version active en circulation. On voit que le service répond encore avec la version précédente, ce qui confirme que le trafic n’a pas encore été basculé.

### Commande de promotion utilisée

![Promotion vers la nouvelle version](docu2/image4.png)

```bash
kubectl argo rollouts promote taskflow -n taskflow
```

Cette commande a été utilisée pour promouvoir la version prête vers le service actif.

### Post-migration : sortie du script `observe.sh`

![Post-migration observe](docu2/image6.png)

Après la promotion, le trafic est maintenant servi par la nouvelle version. La sortie montre la réponse du service après le basculement, ce qui valide que la migration est effective.

### Passage effectif vers la nouvelle version

![Version effective dans le manifeste](docu2/image3.png)

Après promotion, la nouvelle version est activée et on peut le vérifier dans le manifeste : la version de travail et la cible sont bien visibles dans la configuration; la bascule est maintenant effective.

## Canary Part

### Remplacement du `Deployment` par un `Rollout` Canary

![Rollout Canary dans le dépôt](docu2/image7.png)

Le manifest de déploiement est remplacé par un `Rollout` afin d’autoriser les étapes de promotion progressive, le traffic split et la gestion de la fenêtre de validation avant basculement complet.

### PR sur la version `1.1.0`

![Sync Argo CD et statut de l’application](docu2/image8.png)

La PR met à jour l’image vers `1.1.0`. Argo CD synchronise l’application et confirme le bon état global du cluster : le dépôt reste la source de vérité et le cluster est aligné dessus.

### Canary en pause avant promotion

![Canary en pause](docu2/image9.png)

Le rollout est mis en pause pendant la validation. On observe que la nouvelle révision est présente, mais que le trafic n’est pas encore complètement basculé vers elle et que les anciennes révisions restent encore stables.

### Promotion partielle du canary

![Promotion partielle du canary](docu2/image10.png)

La promotion avance progressivement. Les anciennes et nouvelles révisions coexistent temporairement, et le système répartit le trafic de manière contrôlée au lieu d’un basculement brutal.

### Promotion jusqu’à 100 %

![Canary finalisé](docu2/image11.png)

La nouvelle version devient stable après validation. Les anciennes révisions sont alors mises à l’échelle vers zéro et le rollout est considéré comme healthy.

### Tentative de passage en 2.1.0 et Vérification avec `observe.sh`

![Réponses HTTP après promotion](docu2/image12.png)

Le script confirme que le service répond désormais avec `version=2.0.0` et des codes HTTP `200`, ce qui valide la fin du canary à la suite de l'abort. La version 2.1.0 ne fonctionnant pas.

### Blue-Green ou Canary pour TaskFlow ?

Pour un service comme TaskFlow, le meilleur choix dépend du niveau de risque acceptable.

- Le Blue-Green est plus simple à comprendre et à maintenir : un service actif et un service de prévisualisation, avec un basculement net et un rollback rapide. Le coût est faible, mais le risque est plus visible si la version cible est défaillante au moment du switch.
- Le Canary est plus coûteux en operational complexity : il demande plus de surveillance, plus de contrôles de trafic et une validation plus fine. En revanche, il réduit le risque de dégradation globale parce que le trafic est réparti progressivement.

Pour une application de type petite API interne, le Blue-Green est souvent le choix le plus lisible et le plus rapide à exploiter. Pour un service plus critique ou plus sensible au risque de production, le Canary apporte un meilleur contrôle au prix d’une complexité plus élevée.


