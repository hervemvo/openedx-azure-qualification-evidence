# Plan de données AKS / ACA — constat corrigé

Établi le 2026-08-16 par lecture du code, **avant toute dépense**.
**Ce document a été corrigé** : ma première version concluait qu'AKS était bloqué. C'était
faux, et la correction est expliquée en fin de document.

## Le fait de départ

Aucun fichier `.tf` du produit ne provisionne de backend. Vérifié par `grep` sur
l'ensemble du dépôt **et sur tout le répertoire home** (les seuls `.tf` trouvés ailleurs
appartiennent à d'autres projets : genai-landing-zone, azfit, fitness-board).

Les modules provisionnent le calcul, le réseau, l'identité et l'observabilité — jamais
MySQL, MongoDB, Redis, Meilisearch ni SMTP.

## Ce que cela implique — cible par cible

| Cible | Modes de backend autorisés | Conséquence |
|---|---|---|
| **VM** | `embedded` accepté | ✓ qualifiée `READY-WITH-LIMITS` — backends en conteneurs sur la VM |
| **AKS** | **`embedded` accepté** | ✓ **déployable** — Tutor déploie les backends en pods |
| **ACA** | **`external` OBLIGATOIRE** | ✗ **bloqué** — rien ne provisionne les services requis |

La contrainte ACA est explicite dans le code de validation :

```python
# validation.py — ACA requires every source of truth to be external
if runtime_type == "aca" and modes.get(service) not in {"external","managed","s3-compatible"}
if runtime_type == "aca" and modes.get("search") != "external"
```

AKS n'a **aucune** restriction équivalente, ce qui rejoint la conception amont
(`MATRICE-VM-AKS-ACA.md`) : pour MySQL/MongoDB/Redis, AKS est marqué
**« C — embarqué en dev ; externe recommandé prod »**, et la matrice précise que
« les backends de données sont **provisionnés ou référencés** selon leur adaptateur ».

### Vérification faite

Une spec AKS avec `mysql/mongodb/redis/search = embedded` :

```
validate → {"status": "valid"}
plan     → localReady: True · qualificationDeployable: True · blockers: None
```

AKS est donc qualifiable **sans provisionner un seul backend Azure**, et le
dimensionnement tient largement : 2 nœuds `D4as_v5` = 8 vCPU sur un quota de 20
(0 utilisé), quand la VM faisait tourner toute la pile sur **un seul** D4as_v5 à 4,8 %
de CPU sous charge.

## Ce qui bloque réellement AKS

**Un seul point** : `control_repository_url` est une variable **obligatoire sans défaut**,
et le module crée `azurerm_kubernetes_flux_configuration`. Le dépôt `gg-openedx-control`
n'existe pas ; l'étape **E2bis-b (A1-GITOPS) est `todo`**.

C'est un prérequis **prévu au plan**, pas un défaut : AKS-2 dit « déploiement **+ Flux** »
et AKS-3 teste « la dérive Flux ».

## Ce qui bloque réellement ACA

Le plan de données lui-même. ACA impose des sources de vérité externes et rien ne les
provisionne. Il faudrait, avant toute qualification ACA :

| Service | Candidat (table normative) | Nature |
|---|---|---|
| mysql | `azure-database-for-mysql-flexible-server` | Azure |
| mongodb | **`mongodb-atlas`** | **tiers hors Azure** — arbitrage requis (Cosmos DB API MongoDB ?) |
| redis | `azure-managed-redis` | Azure |
| search | `meilisearch-external` | à héberger |
| smtp | `azure-communication-services-email` | Azure (fournisseur enregistré, aucune ressource) |
| objectStorage | `s3-compatible` | à déterminer |

## Défaut de gouvernance — celui-ci reste entier

```
$ gg-openedx plan examples/aks-standard/platform.yaml   # backends 'external'
  qualificationDeployable: True
  blockers: None
```

Une spec déclarant des backends **`external`** est jugée déployable alors qu'**aucune
cible n'est résolue** et que rien ne les provisionne. Même classe que g14 : déclaré sans
être réalisé, et rien ne le signale.

**Correctif recommandé** : rendre le planificateur fail-closed — tout backend `external`
sans point de terminaison résolu doit produire un blocker. C'est ce garde-fou qui manque,
et il protégerait les runs futurs comme celui ajouté pour g14.

## Correction de ma première analyse

Ma version initiale concluait « AKS et ACA ne sont pas déployables ». **C'était une
sur-généralisation** : j'avais lu l'*exemple* `aks-standard` (qui déclare `external`) et
conclu sur la *cible* AKS, sans vérifier que le mode `embedded` lui était ouvert.

La distinction compte : pour ACA la contrainte est **codée en dur** dans la validation ;
pour AKS elle n'existe pas. Lire un exemple n'est pas lire une capacité.

## Où cela mène

- **AKS** : déployable dès E2bis-b (dépôt GitOps) fait, avec backends embarqués.
- **ACA** : bloqué tant que le plan de données n'est pas provisionné et que MongoDB n'est
  pas arbitré.
- **Produit** : le garde-fou fail-closed manque, indépendamment des deux cibles.
