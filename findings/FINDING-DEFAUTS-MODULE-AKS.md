# Défauts du module AKS révélés par le premier déploiement réel

Neuf défauts produit, tous **préexistants** — aucun n'a été introduit pendant ce déploiement.
Ils partagent un trait : ni `tofu validate`, ni les tests unitaires, ni le garde-fou de
réalisation ne pouvaient les voir. Ce sont des refus du service Azure, des dérives, des
inerties ou des chaînes jamais reliées, que seul un appel réel expose.

*(Document complété le 17/08 au soir : D6 et D10 vivaient dans les manifestes v007 et
DESTROY-RESULT sans avoir été remontés ici — écart relevé par l'analyse PROMPT-05.)*

## Le motif commun

Le module n'avait **jamais été appliqué contre Azure**. Il était syntaxiquement valide,
couvert par des tests, conforme au contrat — et indéployable. La validation statique confirme
qu'un code est *bien formé*, jamais qu'il est *acceptable pour le service*.

## Les neuf défauts

| Id | Défaut | Ce qu'Azure ou Flux répondait |
|---|---|---|
| D1 | ACR en `Standard` avec accès public désactivé | `can only be disabled for a Premium Sku` |
| D2 | `local_account_disabled` sans intégration Entra ID | `can only be set on Azure AD integration enabled cluster` |
| D3 | Sortie restreinte au 443 + GitOps en SSH sur le 22 | `dial tcp …:22: connect: connection timed out` |
| D4 | `ssh_known_hosts` jamais câblé | vérification d'identité du serveur impossible |
| D5 | `upgrade_settings` non déclaré | dérive permanente à chaque plan |
| D6 | Durcissement sans remise de clé : cluster inadministrable | `User does not have access to the resource in Azure` |
| D7 | Énumération inerte `control_repository_auth` | `https`/`none` déclarés, seul `ssh` réalisé |
| D8 | Identité de charge : sujet fédéré inventé | aucun jeton projeté, aucun Secret créé |
| D10 | ContainerInsights auto-créé hors état | destroy refusé : « still contains Resources » + 409 |

**D1 et D2** bloquaient la création : le module ne pouvait produire aucun cluster.

**D3** est le plus intéressant. Le module offre `outbound_policy = "restricted"` *et* un GitOps
SSH. Chacun est correct isolément ; ensemble, ils s'excluent. Un défaut de **composition**, non
d'implémentation — le genre qu'aucun test de module pris isolément ne trouve.

**D5** aurait fait échouer la porte `g22` (absence de dérive) à chaque exécution, sans lien avec
aucune action. Une porte qui échoue pour une raison structurelle, pas pour un incident.

## Le correctif D3 mérite d'être noté

Deux issues existaient : ouvrir le port 22 vers Internet, ou garder la sortie fermée. J'ai retenu
`ssh.github.com:443` — GitHub expose SSH sur le port 443 précisément pour les réseaux verrouillés.
Aucun port ouvert, aucun secret créé, la posture contractuelle intacte.

Ouvrir le 22 aurait « réparé » Flux en vidant `restricted` de son sens. Le contrat aurait continué
d'afficher une sortie restreinte qui ne l'était plus.

## L'angle mort du garde-fou

`control_repository_auth` accepte `ssh`, `https`, `none` — mais **seul `ssh` produit un artefact**.
`https` et `none` donnent le même résultat. C'est le motif d'inertie exact contre lequel le
garde-fou de réalisation a été écrit.

Il ne l'a pas vu, et il ne pouvait pas : il teste les champs du **contrat** (`platform.yaml`),
jamais les **variables de module**. La dette d'inertie n'était donc pas à zéro — elle était à zéro
*dans le périmètre mesuré*, ce qui n'est pas la même affirmation.

**À traiter après la qualification :** étendre le garde-fou aux énumérations de variables de
module, puis soit câbler `https`, soit retirer les valeurs non réalisées de l'énumération.

## D6 — Le durcissement sans remise de clé

Le module désactive les comptes locaux **et** active Azure RBAC — sans attribuer le moindre
rôle Kubernetes à quiconque. Le cluster naissait inadministrable, y compris pour l'identité
qui venait de le créer (constaté au premier `kubectl` : `Forbidden` sur tout, manifeste v007).

Correctif : variable `cluster_admin_object_ids` + attribution du rôle « Azure Kubernetes
Service RBAC Cluster Admin », portée au cluster seul. La leçon de conception : **un
durcissement n'est complet que s'il inclut la remise de clé** — verrouiller sans désigner
de porteur de clé ne produit pas de la sécurité, produit une panne d'administration.

## D8 — L'identité de charge : déclarée, jamais reliée

Le module pose `workload_identity_enabled = true` et crée un identifiant fédéré dont le sujet est :

```
system:serviceaccount:openedx-ggqaks:openedx-runtime
```

Quatre maillons manquent pour que cette chaîne existe :

| Maillon | Attendu | Réel |
|---|---|---|
| Espace de noms | `openedx-ggqaks` | `openedx` (celui de Tutor) |
| ServiceAccount | `openedx-runtime` | `default` |
| Annotation `client-id` | sur le ServiceAccount | absente |
| Étiquette `workload.identity/use` | sur les pods | absente |

L'espace de noms est le maillon le plus révélateur. Le module le **fabrique** par
`"openedx-${var.name}"`, tandis que le nom réel vient de la kustomization de Tutor. Deux sources
indépendantes, jamais rapprochées — le module ne pouvait pas tomber juste, et rien ne le signalait.

Aucun ServiceAccount `openedx-runtime` n'est créé nulle part : ni par le module, ni par le dépôt
GitOps. L'identifiant fédéré désigne donc un sujet qui n'existe pas.

**Correctif structurel :** le module doit *recevoir* l'espace de noms d'exécution en entrée au
lieu de l'inventer. Un identifiant fédéré dont le sujet est déduit d'une convention de nommage
locale ne peut pas s'accorder avec un espace de noms décidé ailleurs.

## D9 — Blocage d'attachement des volumes

Indépendant des précédents. Les PVC sont en `ReadWriteOnce` ; les anciens pods, bloqués mais
vivants, retiennent les volumes. Les nouveaux pods échouent en `Multi-Attach error`. Le
déploiement ne peut pas se dérouler tant que les anciens ne sont pas retirés.

Ce n'est pas un défaut du produit : c'est la conséquence d'un déploiement resté coincé assez
longtemps pour que les remplacements s'empilent.

## D10 — ContainerInsights : la ressource que personne n'a déclarée

Activer la supervision (`oms_agent`) fait créer par Azure, automatiquement, une solution
`ContainerInsights(log-…)` dans le groupe de ressources — **hors état Tofu**. Au destroy :
le fournisseur refuse de supprimer un groupe non vide, et le déprovisionnement du
fournisseur `Microsoft.KubernetesConfiguration` répond 409 tant que des ressources existent.
Constaté au g23 : 23/25 détruites, puis purge manuelle de l'orphelin, puis achèvement
(`DESTROY-RESULT-aks.yaml`).

Correctif à porter au module : purger la solution avant le groupe (destroy provisioner ou
consigne d'exploitation), ou l'adopter dans l'état dès la création. En attendant, la
procédure est documentée dans la preuve de destruction.

## Découverts par le garde-fou étendu (P1, 17/08 au soir)

Le garde-fou promis au manifeste v006 — étendre la détection d'inertie aux variables de
module — a trouvé deux défauts de plus **à sa première exécution**.

### D11 — `network_exposure` : deux profils sur trois sont le même

Le module AKS ne compare `network_exposure` qu'à `lab-public`. Conséquence : `private` et
`public-web-private-admin` produisent **une infrastructure identique** — le second promet
pourtant un web public avec administration privée, et le module ne construit aucun ingress.
Même famille que D7 : une valeur d'énumération déclarée, validée, documentée… et inerte.
À réaliser (ingress + exposition différenciée) ou à retirer de l'énumération tant que ce
n'est pas construit.

### D12 — `profile` : `dev`, `small` et `standard` sont indistinguables

Sur les quatre adaptateurs (vm, aks, aca, aca-preview), seule la valeur `ha` change quelque
chose (zones, réplicas, données strictes). `dev`, `small` et `standard` traversent la
validation puis ne pilotent **rien** : le dimensionnement vient des variables explicites
des cibles. Le profil du contrat — censé « déterminer » le gabarit — est décoratif sur
trois valeurs sur quatre. Défaut transversal, pas propre à AKS.

Les deux sont inscrits comme dette dans le garde-fou (`KNOWN_INERT_DEBT`) avec renvoi ici :
le test reste vert parce que la dette est **relue et datée**, pas parce qu'elle est invisible.

## Ce que ce déploiement a coûté et rapporté

Le cycle complet a demandé onze applies (v002→v011) et une campagne de qualification (v016).
Un seul obstacle attrapé avant dépense (l'enregistrement du fournisseur), par vérification
préalable. Tous les autres n'ont été révélés que par l'appel réel.

La leçon est réutilisable : pour ce type de module, **le premier apply réel est un test**, pas une
formalité. Il devrait être budgété comme tel dans toute future cible.
