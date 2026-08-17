# Qualification Open edX sur Azure — dossier de preuves

*Evidence file from a real production-qualification campaign: two Azure runtimes (VM and
AKS) taken through a 23-gate binary matrix, under a dual-validation protocol with
sha256-bound authorization manifests. Content in French; findings speak for themselves.*

---

Ce dépôt publie les preuves d'une campagne réelle de qualification de production :
une plateforme Open edX pilotée par contrat, déployée puis qualifiée sur **deux
runtimes Azure** (machine virtuelle, puis AKS), sous un protocole strict, jusqu'à
la **destruction vérifiée sans orphelin**.

Ce n'est pas un tutoriel. C'est le dossier tel qu'il a été produit, expurgé des
seuls identifiants d'infrastructure.

## Ce que ce dossier démontre

**La règle centrale** : *un contrôle non exécuté est BLOCKED, jamais PASS par absence
de test.* Elle s'applique aux 22 portes de qualification (`qualification/`), et elle
s'est appliquée à l'outillage lui-même — plusieurs faux verts d'outillage ont été
attrapés par des contrôles de second niveau, et sont documentés ici plutôt qu'effacés.

**Le constat qui justifie la méthode** : le module d'infrastructure était
syntaxiquement valide, couvert par des tests, conforme à son contrat — et
**indéployable**. Douze défauts documentés (`findings/`), dont aucun n'était visible
par la validation statique. Un seul a été attrapé avant dépense. D'où la règle :
*le premier apply d'un module jamais appliqué est un test, pas une formalité.*

**Le méta-motif** : *déclaré ≠ réalisé ≠ prouvé*. Un contrat appliqué à l'entrée
(refus des combinaisons interdites) et ignoré à la sortie (2 champs sur 11
réellement réalisés) ; une identité de charge dont le sujet fédéré désignait un
compte de service que rien ne créait ; des options d'énumération que rien ne
distingue. Chaque occurrence est consignée avec sa preuve et son correctif.

## Contenu

| Répertoire | Contenu |
|---|---|
| `findings/` | Les 12 défauts documentés (D1–D12) + les audits de couverture |
| `qualification/` | La matrice des 22 portes + g23, et sa déclinaison AKS |
| `verdicts/` | VM : `NOT-READY` (run 1) → `READY-WITH-LIMITS` (run 2). AKS : 20 PASS / 0 FAIL, destruction prouvée, verdict final suspendu à la seule porte de coût |
| `manifests/` | Une sélection des manifestes d'autorisation consommés — y compris ceux qui documentent **nos propres erreurs** (rotation de secrets après exposition, faux verts) |

## Ce que ce dossier ne prétend pas

- Il ne certifie rien au sens réglementaire — les verdicts sont techniques, bornés
  au périmètre observé, et chaque limite est écrite dans le verdict lui-même.
- Les mesures citées (50 RPS à 100 %, RTO 4 s vérifié au niveau base, restauration
  591=591 tables) valent pour la cible jetable décrite, pas pour un autre système.
- Les identifiants d'infrastructure (coffre, socle) sont expurgés ; les empreintes
  sha256 d'abonnement/locataire sont conservées telles quelles — ce sont des hachés.

## Pourquoi publier les erreurs

Trois expositions de secrets (toutes tournées, anciennes versions désactivées),
des faux verts d'outillage, des consignations corrigées a posteriori : tout y est.
Un processus de qualification vaut par sa capacité à attraper ses propres fautes
— c'est précisément ce que ce dossier documente.

---

**Auteur** : Edgard Hervé Mvogo — [geekguru.tech](https://geekguru.tech) ·
Senior Cloud & AI Platform Architect. Contexte, méthode et offre : voir
[geekguru.tech/preuves](https://geekguru.tech/preuves/).
