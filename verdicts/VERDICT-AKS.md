# Verdict cible AKS — run `r20260816T081939Z-fd38a56d`

**Statut : `QUALIFIED-PENDING-G21`** (etabli le 2026-08-17T18:37:36Z)

## Portes
| Verdict | Nombre | Portes |
|---|---|---|
| PASS | 20 | g01–g19 (sauf g20), g22 |
| NOT-APPLICABLE | 1 | g20 (decision utilisateur du 17/08 — aucune version anterieure a migrer) |
| EN ATTENTE | 1 | g21 (ingestion de facturation incomplete, releve final J+1 — voir `G21-COST.yaml`) |
| g23 | DESTROYED | preuve d'absence complete (`DESTROY-RESULT-aks.yaml`) |

## Regle appliquee
`verdictRules.readyWithLimits` exige g01..g22 regles (PASS ou N-A justifie) **et** g23 DESTROYED.
21 portes sur 22 sont reglees. **Aucun verdict final n'est prononce sur une porte en attente** —
c'est la regle du projet, elle vaut aussi quand il ne manque qu'une porte.

## Verdict attendu a la cloture de g21
`READY-WITH-LIMITS`, avec les limites suivantes deja consignees :
1. **Charge (g17)** mesuree sur l'endpoint de disponibilite (50 RPS, 100 % de 200, p99 249 ms),
   pas sur un parcours authentifie.
2. **Images amont** (overhangio, mysql, mongo…) epinglees par tag et digest, mais non signees
   par nous (`G01-IMAGE-DIGESTS.txt`).
3. **Derive g02/g22** corrigee EN COURS de run (bloc api_server_access_profile) — les deux plans
   vierges datent d'apres le correctif.
4. **Sorties brutes** de certaines mesures de campagne consignees en resume (matrice, manifestes)
   sans fichier brut rejouable — limite du dossier de preuve, relevee par PROMPT-05.

Si le cout final contredit l'estimation sans explication, le verdict sera reevalue sur g21.
