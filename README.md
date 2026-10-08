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

### Abort du canary : l’état passe en `Degraded`

![Abort du canary et dégradation](docu2/image13.png)

Lorsqu’une nouvelle version ne respecte pas les critères de qualité attendus, l’abort du canary permet de revenir immédiatement à un état stable. La sortie montre bien le passage en état `Degraded` et confirme que le système a été ramené à un niveau de service sûr avant de poursuivre ou de corriger la version.

### Tentative de passage en 2.1.0 et Vérification avec `observe.sh`

![Réponses HTTP après promotion](docu2/image12.png)

Le script confirme que le service répond désormais avec `version=2.0.0` et des codes HTTP `200`, ce qui valide la fin du canary à la suite de l'abort. La version 2.1.0 ne fonctionnant pas.

### Blue-Green ou Canary pour TaskFlow ?

Pour un service comme TaskFlow, le meilleur choix dépend du niveau de risque acceptable.

- Le Blue-Green est plus simple à comprendre et à maintenir : un service actif et un service de prévisualisation, avec un basculement net et un rollback rapide. Le coût est faible, mais le risque est plus visible si la version cible est défaillante au moment du switch.
- Le Canary est plus coûteux en operational complexity : il demande plus de surveillance, plus de contrôles de trafic et une validation plus fine. En revanche, il réduit le risque de dégradation globale parce que le trafic est réparti progressivement.

Pour une application de type petite API interne, le Blue-Green est souvent le choix le plus lisible et le plus rapide à exploiter. Pour un service plus critique ou plus sensible au risque de production, le Canary apporte un meilleur contrôle au prix d’une complexité plus élevée.

# LAB J3

## A. Étalon et analyse de robustesse

### Baseline 2.0.0 avec `charge.sh`

![Charge.sh sur la production stable](docu2/image14.png)

Avant d’introduire une révision problématique, nous avons mesuré la version stable `2.0.0` avec `./scripts/charge.sh http://taskflow`. La sortie est utilisée comme point de comparaison : on note les erreurs HTTP, le p95 et le niveau global de qualité de service. C’est la référence qui permettra de montrer qu’une révision suivante a dépassé le seuil acceptable.

### Ajout des fichiers de robustesse

![Ajout des fichiers robustesse](docu2/image15.png)

Le dossier `exemples/robustesse` apporte les composants nécessaires à l’analyse automatique : le `Rollout` dédié, les services, le `ConfigMap` de charge et les templates d’analyse. Ce mécanisme transforme le déploiement simple en un pipeline de validation qualité, avec décision automatique de promotion ou d’abort.

### Vérification des CRD et des ressources associées

![Vérification des CRD et des ressources](docu2/image16.png)

Le manifeste de l’application est désormais un `Rollout`, et non plus un `Deployment`. Avant de lancer l’incident, il faut vérifier que les ressources de Argo Rollouts sont bien présentes dans le cluster : la CRD, le `AnalysisTemplate`, le `ConfigMap` et le `Service`. Ce contrôle permet de confirmer que le mécanisme d’analyse et de promotion automatique est bien branché sur le cluster.

## B. L’incident 2.1.0 et les preuves

### Preuve 1 — le pod post migration est en service et la version est active

![Pods post migration 2.1.1](docu2/image17.png)

Après la mise en place de la version `2.1.1`, on vérifie l’état des pods et leur disponibilité. Cette capture montre que la révision est bien active et que les instances du service sont en cours d’exécution. C’est la première preuve que la nouvelle version est bien déployée et que le système est dans un état observable.

### Preuve 2 — k6 signale des checks en échec

![Résultat k6 avec checks failed](docu2/image18.png)

Le job k6 montre explicitement des checks en échec. La preuve importante ici est le taux d’erreur et la violation des seuils de validation. Cela indique que la métrique de charge n’est plus conforme et que la version n’est plus acceptable pour une promotion continue.

### Preuve 3 — `describe analysisrun` confirme la validation automatique

![Describe AnalysisRun](docu2/image19.png)

La commande `kubectl -n taskflow describe analysisrun <nom>` montre le détail du run. On vérifie notamment que le job k6 a été créé, exécuté et terminé avec un état `Completed`, ainsi que le nom du job associé. Cette preuve confirme que le mécanisme d’analyse automatique a bien surveillé la version et a conduit la validation jusqu’à son terme.

### Preuve 4 — chronologie et fin du rollout

![Événements et Rollout completed](docu2/image20.png)

La commande `kubectl -n taskflow get events --sort-by=.lastTimestamp` montre la séquence de l’incident. On voit le lancement du rollout, la création des pods, l’éxécution du job de validation et enfin le point où le système conclut par `Rollout completed` ou par des événements de validation. Ce qu’on en tire est crucial : la chronologie confirme que la décision de promotion ou d’arrêt vient bien du mécanisme Argo Rollouts, et non d’une opération manuelle. L’ensemble des preuves converge vers la même conclusion : la version évaluée ne respecte pas les critères de qualité attendus, et le système interdit sa progression sans intervention humaine.

## Conclusion J3

Le LAB J3 montre que le déploiement automatisé ne se limite pas à l’application des manifests. Il repose aussi sur la capacité du cluster à mesurer, valider et décider. Les `AnalysisRun`, les jobs k6, les `ConfigMap` et les `Service` sont les briques qui permettent de transformer un simple rollout en mécanisme de sécurité opérationnelle : on ne passe à la version suivante que si elle passe les seuils imposés par l’analyse.


