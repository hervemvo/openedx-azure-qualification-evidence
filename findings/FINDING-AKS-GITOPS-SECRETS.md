# Constat bloquant — AKS/GitOps : les manifestes embarqués portent des secrets en clair

Établi le 2026-08-16 en préparant E2bis-b, **avant tout commit et avant toute dépense**.

## Le fait

Le rendu Tutor k8s pour AKS en **backends embarqués** (`RUN_MYSQL/MONGODB/REDIS/
MEILISEARCH/SMTP=true`) produit des manifestes contenant des secrets **en valeur
littérale** :

```yaml
- name: MEILI_MASTER_KEY
  value: "<clé en clair>"
- name: MYSQL_ROOT_PASSWORD
  value: "<mot de passe en clair>"
```

**`secretKeyRef` : 0 occurrence.** Rien n'est référencé depuis un Secret Kubernetes.
Le mot de passe MySQL de `config.yml` se retrouve tel quel dans `env/k8s/deployments.yml`.

## Pourquoi c'est bloquant pour GitOps

Flux réconcilie **depuis un dépôt Git**. Y placer ces manifestes reviendrait à committer
des identifiants de base de données en clair — ce que ta propre conception interdit
explicitement (`STRUCTURE-DEPOT-GITHUB.md`, section 3) :

> Interdits dans ce dépôt : **valeurs de secrets** ou exports Key Vault ; config.yml
> Tutor rendu ; tfstate ; données apprenant ; clés privées…

C'est aussi la classe d'incident que le gate **g15** existe pour attraper.

## Pourquoi le problème n'était pas apparu avant

Le rendu d'E1.5 utilisait le profil **`external`** : `RUN_*=false`, donc **aucun backend**
dans les manifestes, donc **aucun secret de backend**. Il était committable.

Le passage aux backends **embarqués** — la seule voie qui rendait AKS déployable sans
provisionner de plan de données — réintroduit les backends **et leurs identifiants**.

Autrement dit : les deux voies AKS butent chacune sur un mur différent.

| Voie AKS | Mur |
|---|---|
| `external` (prod) | plan de données non provisionné (aucun `.tf`) |
| `embedded` (dev) | secrets en clair dans les manifestes → non committables |

## Le méta-motif, encore une fois

La conception dit la bonne chose. La provenance d'E1.5 l'énonce noir sur blanc :

> `policy: ... configMaps applicatifs fournis a l'execution via Key Vault CSI/runtime
> (aucun secret dans Git)`

L'intention est correcte et documentée. **L'implémentation ne la réalise pas** — comme
pour g14 (8 clés déclarées, 2 appliquées) et comme pour le plan de données AKS/ACA
(backends `external` déclarés, aucun provisionné).

Troisième occurrence du même écart entre le déclaré et le réalisé.

## Ce qu'il faut pour débloquer

Par ordre de fidélité à la conception existante :

1. **Key Vault CSI + `secretKeyRef`** — voie déjà retenue en E1.5. Les manifestes doivent
   référencer des Secrets Kubernetes alimentés par le driver CSI depuis Key Vault, au lieu
   d'inliner les valeurs. Nécessite un patch Tutor ou un overlay Kustomize.
2. **SOPS / Sealed Secrets** — chiffrer les secrets dans Git. Plus simple, mais introduit
   une clé de déchiffrement à gérer, et s'éloigne de la conception Key Vault.
3. **External Secrets Operator** — équivalent fonctionnel au point 1, autre outillage.

## Recommandation

Ne pas créer le dépôt de contrôle tant que le point 1 n'est pas implémenté : un dépôt
GitOps dont le premier commit contient des identifiants en clair est un incident de
sécurité, pas une étape de projet. L'historique Git conserverait les secrets même après
correction.

**Coût de ce constat : 0 €.** Obtenu par vérification avant commit, pas par incident.
