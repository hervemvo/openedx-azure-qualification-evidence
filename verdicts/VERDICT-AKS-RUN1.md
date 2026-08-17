# RETIRE le 2026-08-17 — l'utilisateur a ordonne la poursuite vers READY.
# Les FAIL g02/g22 ont ete corriges a la racine (bloc dynamique) : 2 plans vierges consecutifs.

# VERDICT AKS — PASSAGE 1 : NOT-READY

**Run** : r20260816T081939Z-fd38a56d · **Clôture** : 2026-08-17T14:07:01Z · Bilan : **PASS 2 · FAIL 2 · BLOCKED 18 · PENDING 1 (g23)**

## Pourquoi NOT-READY, et pourquoi c'est le bon moment de fermer

`g02` et `g22` sont FAIL pour une cause **structurelle** (dérive permanente d'un attribut du
fournisseur) : READY était mathématiquement hors d'atteinte pour ce passage, quel que soit le
nombre de portes mesurées en plus. Fermer maintenant, corriger à froid, requalifier — c'est le
motif exact du passage VM (run 1 NOT-READY → run 2 READY-WITH-LIMITS).

## Ce que ce passage a PROUVÉ (et qui n'existait pas avant)

1. **Le chemin complet fonctionne de bout en bout** : contrat → module Tofu → Azure → Flux →
   GitOps → identité de charge → Key Vault CSI → configuration → base initialisée →
   **Open edX répond 200 sur Kubernetes** (première fois du projet, 17/08 11h35 UTC).
2. **Neuf défauts préexistants du module AKS trouvés et corrigés** — aucun visible par
   `tofu validate` ; seul l'apply réel les révèle. Le module n'avait jamais rencontré Azure.
3. La posture de sécurité mesurée tient : g03 PASS sur 9 contrôles.

## Dettes consignées pour le passage 2

- Dérive `virtual_network_integration_enabled` (cause des 2 FAIL) — piste : lifecycle/version fournisseur.
- La cible n'a jamais consommé la **section expérience du contrat** (g14) — recâbler cible → contrat.
- Invariant manquant : noms de variables INJECTÉS vs LUS (le renommage silencieusement cassant).
- Énumération inerte `control_repository_auth` (https/none sans effet).
- Décisions à trancher avant mesure : g19 (méthode de sauvegarde), g20 (statut).
- ACA : composition jamais testée contre Azure — hors périmètre de ce passage, explicitement différé.

## Incidents de session (hors plateforme)

Trois expositions de secrets dans la transcription (jamais dans les preuves) — toutes tournées,
versions compromises désactivées (v008, v010). Cause commune corrigée : plus aucune valeur de
secret ne transite par une ligne de commande shell.

Verdict final du passage après g23 : destruction et absence d'orphelins à constater.
