# Postmortem — Image 2.1.0 abandonnée automatiquement par l'analyse k6

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 15/10/2026, de 11:55:50 à 11:56:52 (heure de Paris) |
| Version en cause | `taskflow:2.1.0` (ReplicaSet `taskflow-df976ccb5`, révision 6) |
| PR à l'origine | #11 « feat: taskflow 2.1.0 (canary avec analyse automatique) », fusionnée à 11:55:05 |
| Durée d'exposition | environ 1 minute (11:55:55 à 11:56:52) |
| Part du trafic touché | environ 25 % (1 pod sur 4 ; estimation : sans maillage de service, la part de trafic suit la proportion de pods) |
| Détecté par | test de charge k6 (AnalysisRun `taskflow-df976ccb5-6-2`) |
| Résolu par | abandon automatique du Rollout (11:56:52), puis `git revert` par la PR #12 (12:29:21) |

## Contexte

La même image 2.1.0 avait déjà échoué la veille (PR #7, 14/10 16:50) avec un canary sans analyse automatique : environ 4 requêtes sur 40 en erreur HTTP 500, repérées à la main, abandon manuel (`abort`), puis revert par la PR #8 (15/10 10:13). Cet incident-ci concerne la même version, avec l'analyse k6 en place (PR #10).

## Chronologie

Heure de Paris (UTC+2).

| Heure | Événement |
| --- | --- |
| 11:54:07 | Commit `800542a` : image passée à 2.1.0 |
| 11:55:05 | Fusion de la PR #11 dans `main` |
| 11:55:50 | Argo CD applique ; Argo Rollouts crée 1 pod canary en 2.1.0 |
| 11:55:55 | Le service `taskflow-canary` bascule vers le nouveau pod ; palier 25 % ; pause de 20 s |
| 11:56:15 | Début de l'analyse : Job k6 `test-de-charge-k6.1` lancé |
| 11:56:46 | k6 se termine : seuils dépassés |
| 11:56:48 | Le Job est en échec (`BackoffLimitExceeded`) |
| 11:56:52 | `MetricFailed`, `AnalysisRunFailed`, `RolloutAborted` ; pod 2.1.0 supprimé ; le service canary revient vers les pods stables |
| 12:27:21 | Commit de revert `c981645` |
| 12:29:21 | Fusion de la PR #12 : Git redit 2.0.0, le Rollout redevient sain |

## Composant défaillant et cause racine

**Composant défaillant :** l'application TaskFlow en version 2.1.0 (le pod canary). Preuves :

1. `kubectl argo rollouts get rollout taskflow -n taskflow` : `RolloutAborted: Rollout aborted update to revision 6 ... Metric "test-de-charge-k6" assessed Failed due to failed (1) > failureLimit (0)`. `kubectl get rs -n taskflow -o wide` : la révision 6 porte l'image `taskflow:2.1.0`.
2. `kubectl -n taskflow describe analysisrun taskflow-df976ccb5-6-2` : métrique `test-de-charge-k6` en `Failed`, de 09:56:15Z à 09:56:52Z, cible `http://taskflow-canary`.
3. `kubectl -n taskflow logs job/6cd4697c-eedd-49d1-8b50-6f773c91fe77.test-de-charge-k6.1` : p95 = 310,32 ms (seuil 250 ms) et 27,33 % d'erreurs (82 sur 300, seuil 2 %). Référence de la 2.0.0 : p95 = 5,77 ms et 0,00 % d'erreurs (0 sur 735).
4. `kubectl -n taskflow get events --sort-by=.lastTimestamp` : `AnalysisRunRunning`, `MetricFailed`, `AnalysisRunFailed`, `RolloutAborted`, dans cet ordre.

**Pourquoi les probes Kubernetes ne l'ont-elles pas vu ?** Les sondes du Rollout ne testent que `/health`, alors que le test de charge appelle `/tasks`. Le pod canary a été déclaré prêt et a reçu du trafic. Dans cette configuration, une sonde verte ne prouve pas que les utilisateurs sont servis correctement.

**Cause racine :**

- Dans le code de la 2.1.0 : non établie ici, le code applicatif n'étant pas dans ce dépôt. Constat mesuré : une latence presque constante d'environ 300 ms par requête et 27 % de réponses autres que 200 sur `/tasks`.
- Côté processus : rien ne vérifiait le comportement sous charge avant l'exposition, et les sondes ne couvrent pas `/tasks`. De plus, l'abandon agit sur le cluster mais pas sur Git : Git est resté sur 2.1.0 d'environ 11:56 à 12:29, et le Rollout est resté en `Degraded` jusqu'au revert.

## Ce qui a bien fonctionné

- Détection automatique : test de 37 s (11:56:15 à 11:56:52), abandon environ 62 s après le début du déploiement, sans intervention humaine.
- Pendant le canary, 3 pods stables sont restés en service ; le 4ᵉ a été recréé à l'abandon.
- Les 4 preuves (AnalysisRun, Job, logs, événements) ont permis de reconstituer l'incident avec des faits.
- Le retour arrière est passé par une PR, avec trace dans l'historique Git.
- La version suivante, 2.2.0 (PR #13, 14:33), a traversé les mêmes paliers jusqu'à 100 % et réussi le même test (AnalysisRun `taskflow-7ddd57d788-8-2`) : le test ne bloque pas une version saine.

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Corriger la 2.1.0 (latence d'environ 300 ms, 27 % d'erreurs sur `/tasks`) avant toute nouvelle promotion | [à compléter] | [à compléter] |
| Faire vérifier `/tasks` par la sonde de readiness, ou ajouter un endpoint de santé plus complet | [à compléter] | [à compléter] |
| Après un abandon automatique, ouvrir tout de suite la PR de revert (Git reste sur la version défaillante) | [à compléter] | [à compléter] |
| Étendre le test k6 à d'autres endpoints que `/tasks` | [à compléter] | [à compléter] |
