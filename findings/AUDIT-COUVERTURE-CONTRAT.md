# Audit de couverture du contrat — ce que la spec déclare vs ce qu'elle génère

Établi le 2026-08-16. Méthode mécanique, reproductible, coût 0 €.

## Méthode

Pour chaque champ à valeurs énumérées du contrat (`platform-spec-v1alpha1.schema.json`) :

1. rendre la spec de référence (`examples/vm-small`) ;
2. changer **ce seul champ** pour chacune de ses autres valeurs ;
3. re-rendre et comparer les empreintes.

**Point de méthode décisif** : la comparaison ne porte **que sur les artefacts qui
pilotent l'infrastructure** — `tofu.example.tfvars.json`, `platform-inputs.yaml`,
`runtime-plan.json`, templates d'applications.

Une première mesure incluant `platform-plan.yaml` concluait « 11 champs actifs sur 11 ».
C'était faux : ce document **recopie** la spec, donc tout changement d'entrée le modifie
sans rien réaliser. Mesurer l'écho au lieu de l'effet inverse complètement le résultat.

## Résultat : 2 champs actifs sur 11

| Champ | Effet sur l'infrastructure |
|---|---|
| `runtime.type` | ✓ **actif** — sélectionne l'adaptateur |
| `experience.mfes.preset` | ✓ **actif** |
| `endpoints.tls.mode` | ✗ **inerte** (`managed` → `external`) |
| `data.search.mode` | ✗ **inerte** |
| `data.objectStorage.mode` | ✗ **inerte** |
| `data.smtp.mode` | ✗ **inerte** (`external` → `disabled`) |
| `network.exposure` | ✗ **inerte** (`public-web-private-admin` → `private`, `lab-public`) |
| `network.outboundPolicy` | ✗ **inerte** (`restricted` → `allow-required`) |
| `observability.profile` | ✗ **inerte** (`standard` → `minimal`) |
| `backup.profile` | ✗ **inerte** (`daily` → `none`, `continuous`) |
| `delivery.channel` | ✗ **inerte** (`stable` → `candidate`) |

**82 % de la surface énumérée du contrat est décorative.**

## Ce que cela signifie concrètement

Un client peut écrire dans sa spec :

- `backup.profile: none` → il aura quand même des sauvegardes quotidiennes ;
- `network.exposure: private` → il aura exactement la même infrastructure que `lab-public` ;
- `observability.profile: minimal` → il paiera le profil standard ;
- `endpoints.tls.mode: external` → le TLS managé sera provisionné malgré tout.

Le contrat **promet une configuration** et livre une infrastructure **indépendante de
cette configuration**. C'est la cause directe des régressions constatées : l'écart entre
ce qui est déclaré et ce qui est déployé n'est jamais détecté, puisque rien ne le mesure.

## Preuve par le run VM déjà exécuté

Les quatre exemples déclarent `network.exposure: public-web-private-admin`.
La VM qualifiée `READY-WITH-LIMITS` a été déployée avec **0 IP publique et 0 règle NSG
entrante** — donc **entièrement privée**, à l'opposé de sa déclaration.

Personne ne l'a vu, et le verdict de qualification n'en a pas tenu compte : aucun gate ne
confronte la spec déclarée à l'infrastructure obtenue.

## Le motif systémique

Cinquième occurrence du même écart déclaré/réalisé :

| # | Constat | Portée |
|---|---|---|
| 1 | g14 — plugin Tutor : 8 clés déclarées, 2 appliquées | expérience |
| 2 | Plan de données AKS/ACA : backends `external` déclarés, 0 provisionné | infrastructure |
| 3 | « aucun secret dans Git » énoncé en E1.5, non implémenté | sécurité |
| 4 | `network.exposure` inerte | réseau |
| 5 | **9 champs sur 11 inertes** (ce document) | **contrat entier** |

Les quatre premiers étaient des symptômes. Celui-ci est la maladie.

## Correctif structurel

Le correctif n'est pas de réaliser les 9 champs un par un et d'espérer. C'est d'ajouter
un **garde-fou générique** — le test ci-dessus, intégré à la suite :

> pour chaque champ à valeurs énumérées, deux valeurs différentes doivent produire deux
> artefacts d'infrastructure différents ; sinon la CI échoue.

Ce test échouerait aujourd'hui sur 9 champs. Chaque champ réalisé le fait passer au vert,
un par un, et **aucun champ décoratif ne peut plus être ajouté** au contrat.

C'est le même principe que le garde-fou ajouté au plugin Tutor après g14
(`declared_keys() - realized_keys() == ∅`), généralisé au contrat entier.

## Reproduire cet audit

Le script de mesure est déterministe et sans effet de bord (rendus en répertoires
temporaires, aucun appel cloud). Il doit devenir un test permanent plutôt qu'une
vérification ponctuelle.

---

# Contre-audit : les prompts et le contrat confirment-ils le diagnostic ?

Vérification demandée le 2026-08-16 : confronter le constat aux exigences écrites.

## Les prompts délèguent au contrat

`PROMPT-02-EXECUTION.md`, étape E3 : « Implémente `PlatformSpec` v1alpha1 **conformément
à `CONTRAT-PLATEFORME.md`** ». Les trois prompts ne nomment eux-mêmes aucun des neuf
champs inertes : l'autorité est le contrat.

## Le contrat exige explicitement la réalisation

Section `network` :

> Le profil détermine **ingress, endpoints privés, DNS, restrictions administratives,
> trafic sortant et journaux réseau**.

« Le profil **détermine** » — l'exigence est sans ambiguïté : `network.exposure` doit
produire une infrastructure différente. Il ne produit rien.

Ligne 4 : « **Source de vérité** : fichier YAML versionné dans un dépôt de contrôle. »
Le contrat pose donc la spec comme autorité sur l'infrastructure, pas comme documentation.

Section `backup` : « RPO, RTO, rétention et fréquence d'exercice sont **obligatoires en
production**. Le plan doit **nommer chaque source de vérité, son mécanisme de sauvegarde
et son test de restauration**. »

**Conclusion : la non-conformité est établie par écrit, pas supposée.**

## Nuance importante — la couche de validation, elle, fait son travail

Test des refus imposés par la section 8 du contrat :

| Refus exigé | Implémenté |
|---|---|
| accès administratif ouvert au monde (`adminCidrs: 0.0.0.0/0`) | **oui** |
| sauvegarde désactivée en production | **oui** |
| `autoPromote=true` en production | **oui** |
| exposition `lab-public` en production | **non** |

**3 refus sur 4 fonctionnent.** Le produit sait donc refuser une déclaration dangereuse.

Ce qui manque n'est pas la rigueur — elle est réelle et mesurable — mais le **chaînage**
entre la couche qui valide et la couche qui déploie.

## Diagnostic corrigé

| Couche | État |
|---|---|
| **Validation** (refuser le non conforme) | 3/4 — solide, une lacune (`lab-public` en production) |
| **Réalisation** (traduire la spec en infrastructure) | **2/11 — c'est là qu'est le problème** |

Formulation plus juste que la précédente : le contrat est **appliqué en entrée et ignoré
en sortie**. Quelqu'un a écrit un validateur exigeant ; l'IaC a été écrite séparément avec
des valeurs fixes ; rien ne relie les deux.

Cela explique aussi pourquoi le défaut a survécu à toute la phase locale E0→E1.8 : les
tests vérifiaient que la validation refuse ce qu'elle doit refuser, jamais que le
déploiement reflète ce qui a été accepté.

## Ce que cela change pour le correctif

Rien à reprendre côté validation, sauf la lacune `lab-public`. L'effort porte
entièrement sur la **réalisation** et sur le garde-fou qui la rendra non régressive.
