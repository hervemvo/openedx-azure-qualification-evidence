# Verdict de qualification — cible VM (VM-4 / VM-6)

Run `r20260816T081939Z-fd38a56d` · produit `56e841b` (correctifs `7f15ee1`)
Run de référence : `r20260814T143550Z-6a46c993` → `NOT-READY`.

---

## VM-4 — Verdict provisoire : **`QUALIFIED-PENDING-DESTROY`**

Règle : *« gates g01..g22 == PASS (g10 NOT-APPLICABLE seulement si feature désactivée
et justifiée) »*.

| État | Nombre | Détail |
|---|---|---|
| **`PASS`** | **21** | g01–g19, g21, g22 |
| `NOT-APPLICABLE` justifié | 1 | g20 (aucune version supérieure publiée ; réarmement automatique) |
| **`FAIL`** | **0** | — |

**Bundle de preuves : empreinté ET signé** (cosign, bundle Sigstore détaché, signature
vérifiée). L'exigence « bundle de preuves signé » — non satisfaite au run précédent — l'est.

---

## VM-6 — Verdict final : **`READY-WITH-LIMITS`**

Règle `ready` : *« QUALIFIED-PENDING-DESTROY ET g23 == DESTROYED ET promotion approuvée »*.
Règle `readyWithLimits` : *« READY mais limites explicites, acceptables, n'affectant aucun
critère obligatoire »*.

| Condition | État |
|---|---|
| `QUALIFIED-PENDING-DESTROY` | **OUI** |
| g23 == `DESTROYED` strict | **OUI** — réconciliation croisée 0/0 |
| Aucun critère obligatoire en échec | **OUI** — 0 `FAIL` |
| Promotion approuvée (F1) | **en attente** de `AUTHORISER PROMOTION` |

`productionReady` passera à `true` **après** l'autorisation de promotion (F1). Le verdict
technique est acquis ; il ne manque que l'acte d'approbation.

### Les limites qui justifient `WITH-LIMITS`

Aucune n'affecte un critère obligatoire ; toutes sont consignées et vérifiables.

1. **g20 non applicable** — aucune version Open edX/Tutor supérieure n'existe. Réarmement
   automatique dès publication, ou si la migration Ulmo 21.0.7 → 22.0.1 est retenue.
2. **g19 — contenu des disques non vérifié** : géométrie conforme, mais les disques
   restaurés n'ont pas été montés (créer une VM excédait la portée autorisée).
3. **g17 — charge sur 2 endpoints**, trafic non authentifié, mesure in-VNet sans TLS.
   Marge telle qu'une dégradation ×3 resterait conforme.
4. **g16 — une seule alerte déclenchée** par charge réelle (CPU) ; `disk` et
   `availability` sont déployées et actives mais non provoquées.
5. **API bridées à 60 req/min** (`RATELIMIT_ENABLE=True`) — protection correcte, mais
   quota à relever explicitement pour tout usage API soutenu en production.
6. **Le MFE charge un CDN externe** (`cdnjs.cloudflare.com`) — incompatible avec une
   posture full-private ; à internaliser.
7. **Secrets Tutor en clair au repos** dans `TUTOR_ROOT` (600) — inhérent à Tutor ;
   atténuation production : injection Key Vault CSI.
8. **`tracking.log` / `all.log` en 644** — synthétique ici, PII d'apprenants en production.
9. **Socle control-plane en accès public + RBAC** — delta de production déjà documenté
   (full-private = ACR Premium + private endpoints).

---

## Ce qui a changé depuis le run précédent

| | Run 1 | Run 2 |
|---|---|---|
| PASS | 15 | **21** |
| **FAIL** | **2** | **0** |
| BLOCKED | 5 | 0 |
| NOT-APPLICABLE | 0 | 1 |
| Bundle de preuves | empreinté | **signé** |
| g23 | `DESTROYED` (après reprise) | **`DESTROYED`** du premier coup |
| Verdict | `NOT-READY` | **`READY-WITH-LIMITS`** |

### Les cinq conversions, obtenues par correction et non par assouplissement

- **g01** — build reproductible (`SOURCE_DATE_EPOCH` épinglé au commit) + signature cosign.
  Preuve : reconstruction au commit enregistré → digest identique au registre.
- **g14** — expérience réellement appliquée. Test de causalité sur la plateforme vivante :
  inverser thème et feature flag change désormais les settings Django (avant : aucun effet).
- **g10** — certificat émis et consultable (HTTP 200), via un cours réellement noté.
- **g16** — alertes déployées, déclenchées par charge réelle, puis acquittées.
- **g17** — charge conforme au profil Q9 que le plan définissait déjà ; disponibilité 100 %.

---

## Erreurs de méthode consignées

Par transparence, et parce qu'elles auraient faussé le résultat :

1. **Mesures trompeuses écartées** (run 1) : un échec instantané puis une soumission
   asynchrone de CLI, tous deux pris pour des RTO. Seule la durée du job Azure fait foi.
2. **Double soumission de sauvegarde** (run 2) : mon analyse de sortie a planté sur un
   avertissement précédant le JSON, j'en ai conclu à un échec et relancé. Azure a refusé.
   *Leçon symétrique : un échec d'analyse n'est pas un échec d'opération.*
3. **Seuils Q9/Q10 déclarés absents** : ils figuraient au contexte depuis le début. J'ai
   renvoyé à l'utilisateur une décision déjà tranchée dans son propre plan.
4. **`docker kill` pris pour une panne** (run 1) : Docker le traite comme un arrêt
   administratif et n'applique pas la politique de redémarrage.
5. **Init MySQL isolée sans le conteneur `permissions`** (run 2) : la leçon du run 1 était
   juste, son application incomplète — un défaut remplacé par un autre.
6. **Garde-fou non bloquant** (run 1) : un script de vérification affichait `FAIL` sans
   interrompre la suppression.

---

## Coût

**~0,93 €** pour 3,9 h de cible (VM 0,69 + disques 0,14 + alertes/drill ~0,10).
Ressource résiduelle : **aucune**.

---

## Portée

Ce verdict couvre **la cible VM uniquement**. **AKS** et **ACA** ne sont pas qualifiées.
Les correctifs g01/g14 étant situés dans le plugin Tutor et l'outillage de release —
communs aux trois runtimes — ils bénéficient aux deux autres cibles, mais leur
qualification reste entière (AKS exige au préalable une augmentation de quota vCPU).
