# Postmortem — la 2.1.0 est passée en production, l'analyse k6 ne l'a pas vue

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 8 octobre 2026, de 11:10 à environ 11:54 (heure de Paris) |
| Version en cause | `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0` |
| PR à l'origine | Première promotion : révision 6, commit `5fccb6a`. Abort automatique : révision 8, PR #19 (`6853b55`) |
| Durée d'exposition | De la fin de l'analyse `df976ccb5-6-1` (11:10) jusqu'au retour des pods `2.0.0`, visible à 11:54 |
| Part du trafic touché | 100 % après la promotion. L'analyse a validé le palier, le poids est monté jusqu'au bout |
| Détecté par | Comparaison des logs k6 avec les variables de l'image, puis le journal du contrôleur Argo Rollouts. Les probes n'ont rien signalé |
| Résolu par | Pause de 15 s avant l'analyse, puis abort automatique de la révision 8. La `2.2.0` (PR #20, `ed69cd1`) a passé l'analyse |

## Chronologie

| Heure | Événement |
| --- | --- |
| 10:49 | `observe.sh` : 40/40 `version=2.0.0` HTTP 200 |
| 10:50 | `charge.sh` sur `http://taskflow`, scénario k6 de 30 s et 5 utilisateurs virtuels |
| 11:00 | Rollout, `AnalysisTemplate` `robustesse-k6`, ConfigMap `k6-robustesse`, Services `taskflow` et `taskflow-canary`. Aucun Deployment |
| 11:10:26 | Pod canary `2.1.0` (`df976ccb5-rcvxh`) démarré. Le contrôleur retarde le basculement de `taskflow-canary` : le ReplicaSet n'a encore aucun pod disponible |
| 11:10:33 | Le pod devient Ready. Dans la même seconde, le sélecteur passe de `c6cf57bd6` (`2.0.0`) à `df976ccb5`, et le Job k6 démarre |
| 11:11:08 | k6 termine : 736 réponses HTTP 200, `http_req_failed` à 0,00 %, p(95) à 6,53 ms. L'`AnalysisRun` `df976ccb5-6-1` est **Successful** |
| ensuite | Le Rollout va jusqu'à 100 %. Les quatre pods sont en `2.1.0`. L'image a pourtant `FAILURE_RATE=0.3` et `LATENCY_MS=300` |
| après la pause de 15 s | Révision 8, PR #19. L'`AnalysisRun` `df976ccb5-8-2` échoue. Message : métrique `test-de-charge-k6` en échec, `failureLimit` à 0. Abort automatique, poids à 0, stable `2.0.0` |
| 11:54 | Argo CD **Degraded** et **Synced** sur `6853b55`. Les pods qui répondent sont ceux de `c6cf57bd6` |
| 12:06 | PR #20 mergée (`ed69cd1`), image `2.2.0` |
| 12:08 | Révision 9, `AnalysisRun` `7ddd57d788-9-2` **Successful** |
| 12:12 | Rollout **Healthy**, pas 7/7, poids 100 %, quatre pods `2.2.0` |

## Composant défaillant et cause racine

L'application `2.1.0` est en panne, et la porte qui devait l'arrêter a mesuré la mauvaise version.

- Le middleware `inject_faults` laisse passer `/health`. Sur `/tasks`, il attend 300 ms puis renvoie un HTTP 500 (`{"detail": "Erreur interne"}`) pour environ 30 % des requêtes. Les variables d'environnement du pod le confirment : `APP_VERSION=2.1.0`, `FAILURE_RATE=0.3`, `LATENCY_MS=300`. Les tags `1.0.0`, `1.1.0` et `2.0.0` ont ces deux derniers à 0.
- Les probes Kubernetes appellent `/health`. Le pod reste `Ready` pendant que les utilisateurs prennent les 500. Le readiness ne peut pas voir cette panne.
- k6, lui, appelle `/tasks` et refuse plus de 2 % d'erreurs ou un p(95) au-dessus de 250 ms. Sur l'`AnalysisRun` `df976ccb5-6-1`, aucun de ces seuils n'a bougé : 0 % d'erreurs, p(95) à 6,53 ms, durée maximale 31 ms. Ce profil est celui de la `2.0.0`, pas de la `2.1.0`.
- Le journal du contrôleur explique l'écart. Il a retardé le changement de sélecteur de `taskflow-canary` tant que le nouveau ReplicaSet avait zéro pod disponible (`delaying service switch from c6cf57bd6 to df976ccb5`). Le basculement et le démarrage du Job sont tombés dans la même seconde. Les cinq connexions de k6 sont restées sur les pods `2.0.0`, encore présents, pendant les 30 s du test.

Cause racine : l'analyse démarre au moment où le Service canary change de pods, et k6 réutilise ses connexions. Le test valide l'ancienne version. Le Rollout prend ce succès pour un feu vert et envoie 100 % du trafic vers une image qui répond 500.

## Ce qui a bien fonctionné

- La deuxième tentative, après une pause de 15 s entre le palier de 25 % et l'analyse, a mesuré la `2.1.0`. L'`AnalysisRun` `df976ccb5-8-2` a échoué et le Rollout a abandonné tout seul la révision 8. Personne n'a lancé `abort`.
- Le ReplicaSet stable `c6cf57bd6` est resté en `2.0.0`, avec quatre pods. Le canary a été réduit à zéro.
- Le retour vers `2.0.0` (révision 7) a d'abord produit un `AnalysisRun` en échec (`c6cf57bd6-7-1`), puis un succès (`c6cf57bd6-7-2`). Le succès a rendu la main à la version saine.
- La `2.2.0` a passé la même analyse (`7ddd57d788-9-2`) et le Rollout est allé jusqu'à 100 %, **Healthy**.

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Pause de 15 s entre `setWeight: 25` et l'analyse, pour que le sélecteur de `taskflow-canary` vise les pods neufs avant l'ouverture des connexions k6. Commit `9de9334` | jabdoulie | Fait le 8 octobre 2026 |
| Garder `duration: 30s`. Allonger le test ne relance pas une analyse déjà réussie, et ne force pas k6 à ouvrir de nouvelles connexions | jabdoulie | Fait |
| Au prochain scénario, ajouter `noConnectionReuse: true` dans le script k6, pour qu'une connexion ouverte trop tôt ne reste pas collée à l'ancien pod | binôme | Prochaine itération du scénario |
