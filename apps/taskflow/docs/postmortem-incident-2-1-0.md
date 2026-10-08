# Postmortem — Incident de validation sur la version 2.1.0

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 2026-10-08 |
| Version en cause | 2.1.0 |
| PR à l'origine | PR #23 — passage vers la version 2.1.0 |
| Durée d'exposition | Environ 10 à 15 minutes, jusqu'à correction du selector et relance du rollout |
| Part du trafic touché | 25 % du trafic durant le canary, avant validation par l'analyse |
| Détecté par | Analyse Argo Rollouts + k6 + observabilité Kubernetes |
| Résolu par | Correction du selector, temps d'attente plus long, puis passage validé vers 2.2.0 |

## Chronologie

| Heure | Événement |
| --- | --- |
| T0 | Mise en place de la version 2.1.0 via PR GitOps |
| T0 + 1 min | Argo Rollouts démarre le canary à 25 % |
| T0 + 2 min | Les nouveaux pods sont créés, mais le selector de service n'est pas encore correctement propagé |
| T0 + 3 min | Les checks k6 semblent échouer, donnant un faux positif de défaillance |
| T0 + 4 min | `AnalysisRun` semble indiquer un échec, mais la cible observée n'est pas encore la bonne |
| T0 + 6 min | Diagnostic : le selector ciblait encore les anciens pods ou les pods non totalement alignés |
| T0 + 8 min | Augmentation du temps d'attente et correction du ciblage du service |
| T0 + 10 min | Validation de l'abort automatique sur les vrais pods ciblés |
| T0 + 12 min | Passage validé vers la version 2.2.0 et stabilisation du rollout |

## Composant défaillant et cause racine

- Le composant concerné n'était pas la logique applicative de la version 2.1.0, mais le mécanisme de ciblage de la validation dans le canary.
- La vraie cause racine est un faux positif lié au selector non encore propagé sur les nouveaux pods : les tests de charge observaient la mauvaise cible pendant la phase d'analyse.
- Les commandes suivantes montrent la nature du problème et la correction :

```bash
kubectl argo rollouts get rollout taskflow -n taskflow
kubectl -n taskflow get pods
kubectl -n taskflow describe analysisrun <nom-du-run>
kubectl -n taskflow get events --sort-by=.lastTimestamp
```

- Observation clé : les `AnalysisRun` et les logs k6 se montraient négatifs uniquement parce que la validation ciblait un état incomplet des nouveaux pods, et non parce que la version 2.1.0 était intrinsèquement cassée.
- Les probes Kubernetes n'ont pas permis de détecter le problème dès le départ parce que la disponibilité des pods était temporairement correcte et le problème était un mauvais alignement de service/selector, plus qu'un crash applicatif ou une panne réseau.
- Cause racine : synchronisation tardive du selector sur les nouveaux pods + temps de validation trop court.

## Ce qui a bien fonctionné

- Argo Rollouts a bien démarré le canary et tenté l'analyse automatique.
- Le mécanisme d'abort était présent et prêt à arrêter le rollout en cas de vrai seuil de défaillance.
- La détection a bien permis d'identifier un problème de validation, même si la première interprétation était fausse.
- La correction du temps d'attente et du ciblage du selector a permis d'obtenir une vraie lecture fiable du système.
- La version 2.2.0 a ensuite été validée correctement après correction du faux positif.

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Vérifier le selector et le service canary avant de conclure à un échec réel | Équipe platform / DevOps | Immédiat |
| Augmenter le temps d'attente avant validation du canary | Équipe platform / DevOps | Immédiat |
| Conserver `abortOnStepFailure: true` pour bloquer tout vrai échec de seuil | Équipe platform | Immédiat |
| Documenter le vrai scénario de faux positif dans le postmortem | Équipe de projet | Avant clôture du lab |
| Valider le passage final vers 2.2.0 avec les captures de preuve | Équipe de projet | Avant livraison du lab |

## Conclusion

L'incident initial a bien montré l'importance du temps de propagation des objets Kubernetes et de la cohérence entre le service canary et les nouveaux pods. Le vrai défaut n'était pas “la version 2.1.0 est cassée”, mais “la validation observait la mauvaise cible pendant la période de propagation”. La bonne correction a permis de restaurer la fiabilité du mécanisme, puis de valider le passage vers 2.2.0 avec succès.
