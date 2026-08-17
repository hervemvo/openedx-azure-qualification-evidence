# Verdict final cible AKS — run `r20260816T081939Z-fd38a56d`

## **`READY-WITH-LIMITS`** — prononcé le 2026-08-17T20:38:43Z

Règle `verdictRules.readyWithLimits` : g01..g22 réglées (PASS ou N-A justifié) ET g23 DESTROYED.

| Verdict | Nombre | Portes |
|---|---|---|
| PASS | 21 | g01–g19, g21, g22 |
| NOT-APPLICABLE | 1 | g20 (décision utilisateur — aucune version antérieure à migrer) |
| DESTROYED | g23 | 25/25 + 1 orphelin auto-créé purgé, preuve d'absence complète |

## Les limites, explicites

1. **g17 (charge)** : 50 RPS × 120 s à 100 % de réussite, p99 249 ms — mesuré sur
   l'endpoint de disponibilité, pas sur un parcours authentifié.
2. **Images amont** épinglées par tag ET digest, non signées par nous.
3. **g02/g22** : la dérive structurelle a été corrigée EN COURS de run ; les deux
   plans vierges datent d'après le correctif.
4. **Dossier de preuve** : certaines mesures de campagne sont consignées en résumé
   (matrice, manifestes) sans sortie brute rejouable dédiée.
5. **g21** : clos sur décision utilisateur avec ingestion incomplète (4,48 € ingérés,
   compteur arrêté par destruction, borne ≈ enveloppe 8-10 €) ; chiffre final annexé
   à `G21-COST.yaml` quand l'ingestion sera stable.

Aucune de ces limites n'affecte un chemin critique de production ; chacune est datée,
mesurée et rejouable. `productionReady` reste dépendant de la promotion (F1).
