# Postmortem — La version 2.1.0 promue en production malgré l'analyse automatique


| Champ | Valeur |
| --- | --- |
| Date et heure | 8 octobre 2026, de 11:03 à 12:08 (deux incidents) |
| Version en cause | `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0` |
| PR à l'origine | #17 (incident 1), #21 (incident 2, redéploiement volontaire pour vérifier l'analyse) |
| Durée d'exposition | Incident 1 : environ 11:04 → 11:26 (≈ 22 min). Incident 2 : environ 11:58 → 12:08 (≈ 10 min) |
| Part du trafic touché | 25 % pendant le canary, puis 100 % du trafic. Entre 25 % et 40 % des requêtes en erreur 500 |
| Détecté par | Un humain, avec `./scripts/observe.sh`. **Pas** par le test de charge automatique |
| Résolu par | `git revert` par PR (#18 puis #22), puis `kubectl argo rollouts promote --full` |

## Résumé

La version 2.1.0 renvoie des erreurs 500 sur une partie des requêtes.
Notre chaîne de déploiement contient un test de charge automatique (k6) pendant le canary,
qui devait bloquer cette version. **Il l'a laissée passer deux fois.**

Le test a réussi (0 % d'erreurs), car **il n'a pas mesuré la version 2.1.0** :
le pod canary 2.1.0 n'a presque pas reçu de requêtes de k6.

Pire : lors du premier retour arrière vers la 2.0.0, le test a mesuré les pods 2.1.0 cassés,
a échoué, et l'abort automatique a **bloqué le retour arrière**. La 2.1.0 est restée en production.

---

## Chronologie

Heures de Paris. Sources : historique Git (heures de merge des PR), événements Kubernetes,
sorties de `observe.sh` et `charge.sh`. Les heures marquées « ≈ » sont calculées à partir
de l'âge des événements Kubernetes.

### Avant l'incident

| Heure | Événement |
| --- | --- |
| ≈ 10:10 | Étalon de la 2.0.0 avec `charge.sh` : 740 requêtes, 0 % d'erreurs, p95 = 3,48 ms |
| 10:43 | PR #15 mergée : analyse automatique k6 ajoutée au canary |
| 10:59 | PR #16 mergée : test k6 allongé à 60 secondes |
| ≈ 11:01 | Nouvel étalon 2.0.0 sur 60 s : 1 480 requêtes, 0 % d'erreurs, p95 = 3,3 ms |

### Incident 1

| Heure | Événement |
| --- | --- |
| 11:03 | PR #17 mergée : image 2.1.0 |
| ≈ 11:04 | Canary 25 % : 1 pod 2.1.0 (`taskflow-df976ccb5-thtkc`). L'analyse k6 démarre |
| ≈ 11:05 | Analyse **réussie** : 1 480 requêtes, 0 % d'erreurs, p95 = 3,9 ms |
| ≈ 11:06 | La 2.1.0 monte à 50 %, 75 %, puis **100 %**. Tous les utilisateurs sont touchés |
| ≈ 11:12 | **Détection humaine** : `observe.sh` → 16 erreurs 500 sur 40 (40 %) |
| 11:18 | PR #18 mergée : revert vers la 2.0.0 |
| ≈ 11:19 | Analyse du revert **échouée** : 29,83 % d'erreurs, p95 = 305 ms |
| ≈ 11:20 | Abort automatique : le trafic reste sur la « stable »… qui est la 2.1.0 |
| ≈ 11:26 | `retry` + `promote --full` : retour à la 2.0.0. `observe.sh` → 40 × 2.0.0, http 200. **Fin de l'impact** |
| 11:29 | PR #19 mergée : version corrigée 2.2.0, déployée sans incident |
| 11:50 | PR #20 mergée : `abortOnFail` ajouté aux seuils k6 |

### Incident 2 (redéploiement volontaire, pour vérifier l'analyse)

| Heure | Événement |
| --- | --- |
| 11:58 | PR #21 mergée : image 2.1.0, avec observation des endpoints du service canary |
| ≈ 11:58 | Les endpoints de `taskflow-canary` contiennent **uniquement** le pod 2.1.0 (`10.244.0.114`) |
| ≈ 11:59 | Analyse **réussie** : 0 % d'erreurs, p95 = 3,44 ms |
| ≈ 12:00 | La 2.1.0 monte à 75 %, puis 100 %. L'`abort` manuel arrive trop tard |
| ≈ 12:02 | `observe.sh` → 12 erreurs 500 sur 40 (30 %) |
| 12:05 | PR #22 mergée : revert vers la 2.2.0 |
| ≈ 12:07 | `promote --full` |
| 12:08 | `observe.sh` → 40 × 2.2.0, http 200. **Fin de l'impact** |

---

## Composant défaillant et cause racine

### 1. Le composant défaillant : l'image 2.1.0

La 2.1.0 renvoie des erreurs 500 sur ses vraies routes (`/` et `/tasks`), sur **tous** ses pods,
y compris le plus récent.

| Preuve | Résultat |
| --- | --- |
| `./scripts/observe.sh` (production en 2.1.0) | 24 × 2.1.0 http 200, **16 × http 500** |
| 40 requêtes sur `/tasks` avec `curl` | 30 × 200, **10 × 500** |
| Comptage des erreurs dans les logs, pod par pod (`docs/preuves/10-erreurs-par-pod.txt`) | Erreurs 500 sur les 4 pods 2.1.0 |

### 2. Pourquoi les probes Kubernetes ne l'ont-elles pas vu ?

Les probes (`readinessProbe` et `livenessProbe`) ne testent que **`/health`**.
La 2.1.0 répond correctement sur `/health`. Kubernetes, Argo Rollouts et Argo CD
la considéraient donc comme saine (`ready:1/1`, `Healthy`).
**Les probes vérifient que l'application démarre, pas qu'elle rend le bon service aux utilisateurs.**

### 3. Cause racine : le test de charge n'a pas mesuré la nouvelle version

C'est ce qui a permis à l'erreur d'atteindre 100 % des utilisateurs.

| Preuve | Ce qu'elle montre |
| --- | --- |
| Logs k6 de l'analyse 2.1.0 (`docs/preuves/3-k6.txt`) | 1 480 requêtes sur `/tasks`, 0 % d'erreurs : le comportement de la 2.0.0, pas de la 2.1.0 |
| Logs du pod canary `thtkc` (incident 1) | **329 requêtes au total**, probes comprises. k6 en a envoyé 1 480 |
| Logs du pod canary `6z5fg` (incident 2, `docs/preuves/9-logs-pod-canary-6z5fg.txt`) | **0 requête `GET /tasks`**. Or k6 n'appelle que `/tasks` |
| Logs k6 de l'analyse du revert (`docs/preuves/3b-k6-revert.txt`) | 29,83 % d'erreurs, p95 = 305 ms : le comportement de la 2.1.0, alors qu'on testait la 2.0.0 |
| Logs des pods 2.1.0 | Erreurs venant de `10.244.0.96`, l'IP du pod k6 **du revert** |

**Conclusion prouvée :** l'analyse automatique a testé la **version stable du moment**
au lieu de la **nouvelle version**. D'où un faux succès pour la 2.1.0,
puis un faux échec pour le retour à la 2.0.0.

### 4. Pourquoi le test a visé les mauvais pods : mécanisme non établi

Nous avons vérifié plusieurs hypothèses. Nous les listons toutes, avec leur résultat :

| Hypothèse | Vérification | Résultat |
| --- | --- | --- |
| Le bug de la 2.1.0 ne touche que la route `/`, pas `/tasks` testée par k6 | 40 requêtes sur `/tasks` en production | **Écartée** : 25 % d'erreurs sur `/tasks` aussi |
| Argo CD (`selfHeal`) annule le sélecteur ajouté par Argo Rollouts sur `taskflow-canary` | Sélecteur du service | **Écartée** : le sélecteur contient bien `rollouts-pod-template-hash` |
| Le service canary n'isole pas la nouvelle version quand un ancien ReplicaSet est réutilisé | Second essai, avec le même ReplicaSet réutilisé | **Écartée** : les endpoints visaient bien le seul pod 2.1.0 |
| La 2.1.0 ne se dégrade qu'après un certain temps | Comptage des erreurs par pod | **Écartée** : le pod le plus récent a aussi des erreurs |
| kube-proxy ne met pas à jour le routage | État et logs de kube-proxy | **Non confirmée** : aucun redémarrage, aucune erreur |
| **Les connexions de k6 restent ouvertes (keep-alive) et accrochées aux anciens pods** | À tester | **Hypothèse restante** |

**Hypothèse restante.** k6 ouvre 5 connexions, une par utilisateur virtuel, et les garde
ouvertes pendant tout le test. Un service Kubernetes choisit le pod **au moment où une connexion
s'ouvre**, pas à chaque requête. Si les connexions de k6 se sont ouvertes juste avant
la bascule du service canary, elles sont restées sur les anciens pods pendant tout le test.
Cette hypothèse explique toutes les observations, mais **elle n'est pas encore prouvée**.

### 5. Ce qui a laissé passer l'erreur

- **Aucune vérification que le test atteint bien la nouvelle version.** Un test réussi était considéré comme une preuve, sans contrôler sa cible.
- **Une seule source de décision** : l'analyse k6. Aucune autre mesure (taux d'erreurs réel des utilisateurs) ne pouvait la contredire.
- **Le retour arrière passait par la même analyse**, et a donc été bloqué par le même défaut.

---

## Ce qui a bien fonctionné

- **Le retour arrière par Git** : chaque revert est une PR tracée, relue et approuvée (#18, #22).
- **La détection humaine** : `observe.sh` a révélé les erreurs en quelques secondes.
- **Les preuves ont été sauvegardées à temps** (`docs/preuves/`), avant la disparition des pods et des événements Kubernetes (conservés environ 1 heure).
- **La méthode d'enquête** : chaque hypothèse a été testée et écartée sur des faits, plutôt que d'accepter la première explication.
- **La 2.2.0** a été déployée et vérifiée sans incident : 1 478 requêtes, 0 % d'erreurs, p95 = 3,28 ms.

---

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Tester l'hypothèse keep-alive : ajouter `noConnectionReuse: true` aux options k6, puis redéployer la 2.1.0 et vérifier que l'analyse échoue | zakariastrong | 15/10/2026 |
| Après chaque analyse, vérifier que le pod canary a bien reçu les requêtes de k6 (compter les `GET /tasks` dans ses logs) | Milhhhhane | 15/10/2026 |
| Ajouter une courte pause (par exemple 15 s) entre `setWeight: 25` et l'analyse, pour laisser le temps au service canary de basculer | zakariastrong | 15/10/2026 |
| Tester plusieurs routes dans le scénario k6 (`/`, `/tasks`, `/health`), pas seulement `/tasks` | Milhhhhane | 22/10/2026 |
| Ajouter une seconde mesure pendant le canary, basée sur le vrai trafic des utilisateurs (taux d'erreurs mesuré par Prometheus) | zakariastrong | 22/10/2026 |
| Documenter la procédure d'urgence : quand l'analyse bloque un revert, utiliser `retry` puis `promote --full` vers la version demandée par Git | Milhhhhane | 15/10/2026 |
| Fait : `abortOnFail` sur les seuils k6, pour arrêter le test dès qu'un seuil est franchi (PR #20) | zakariastrong | 08/10/2026 |

---

## Preuves

Fichiers dans `docs/preuves/` :

| Fichier | Contenu |
| --- | --- |
| `1-rollout.txt` | État du Rollout après l'incident 1 |
| `2-analysisrun.txt` | Détail de l'analyse 2.1.0 (réussie) |
| `2b-analysisrun-revert.txt` | Détail de l'analyse du revert (échouée) |
| `3-k6.txt` | Résumé k6 de l'analyse 2.1.0 : 0 % d'erreurs |
| `3b-k6-revert.txt` | Résumé k6 de l'analyse du revert : 29,83 % d'erreurs |
| `4-evenements.txt` | Événements Kubernetes (chronologie) |
| `6-logs-pods-2.1.0.txt` | Logs des pods 2.1.0 (erreurs 500) |
| `7-pods-2.1.0.txt` | Pods et IP pendant l'incident 1 |
| `9-*.txt` | Analyse, résumé k6 et logs du pod canary pendant l'incident 2 |
| `10-erreurs-par-pod.txt` | Erreurs 500 sur chaque pod 2.1.0 |
| `11-kube-proxy.txt` | Logs de kube-proxy (aucune erreur) |
