# Plan de qualification AKS — déclinaison des 22 portes

Les portes ne changent pas. Ce qui change, c'est **la manière de les mesurer**, parce que
l'exécution passe de « conteneurs sur une VM » à « pods pilotés par Flux ».

Règle inchangée : *un contrôle non exécuté est BLOCKED, jamais PASS par absence de test.*

## Contrainte d'accès qui conditionne toutes les mesures

Le serveur d'API est **privé** et les comptes locaux sont **désactivés**. Il n'existe pas de
kubeconfig administrateur à récupérer. Toute commande passe par :

```
az aks command invoke --resource-group <rg> --name <cluster> --command "kubectl ..."
```

C'est l'équivalent AKS de `az vm run-command` utilisé sur la VM. Conséquence pratique : chaque
mesure est une soumission de tâche, pas une session interactive. J'ai déjà été piégé sur la VM
en confondant *durée de soumission* et *durée d'exécution* — pour AKS, tout chronométrage se
prend **dans** la commande invoquée, jamais autour de l'appel `az`.

## Portes identiques à la VM

`g01` chaîne d'approvisionnement, `g15` absence de secrets et de données personnelles,
`g21` coût observé, `g23` destruction. Mêmes preuves, même méthode.

## Portes dont la mesure change

| Porte | Sur la VM | Sur AKS |
|---|---|---|
| `g02` idempotence | second `plan` sans dérive | idem + **Flux ne doit pas redériver** |
| `g03` identité, secrets, exposition | pare-feu, IP publique | API privé, RBAC Azure, **montage CSI**, identité de charge |
| `g04` installation, migrations | `tutor local` | **réconciliation Flux**, jamais de `kubectl apply` à la main |
| `g13` files et tâches | conteneurs Celery | pods Celery, ordonnancement Kubernetes |
| `g18` reprise après panne | tuer un processus dans le conteneur | **supprimer un pod** — le remplacement est le comportement attendu |
| `g22` absence de dérive | `plan` seul | `plan` **et** état de réconciliation Flux |

## Portes à statuer avant de mesurer

- **`g19` sauvegarde et restauration.** Sur la VM, Azure Backup couvrait la machine entière. Ici,
  les données sont dans le plan de données ; en mode `embedded` elles vivent dans des volumes du
  cluster. Il faut trancher la méthode **avant** de lancer quoi que ce soit — sinon je mesurerai
  une reprise qui ne prouve rien, comme la fausse mesure « 4 s » du premier passage VM.
- **`g20` mise à niveau sur clone.** Écartée en NOT-APPLICABLE justifié sur la VM. Le même
  raisonnement s'applique *a priori*, mais je ne le reconduis pas d'office : à statuer.
- **`g17` charge.** Seuils déjà fixés au contrat (1000 apprenants, 50 RPS). Le cluster est
  volontairement sous-dimensionné pour tenir dans le quota — **il est probable que cette porte
  échoue, et ce sera un résultat honnête**, pas un incident. À consigner comme limite de quota,
  pas comme défaut du produit.

## Ce qui n'a jamais tourné

Les six obstacles rencontrés portaient tous sur la **création** de l'infrastructure. Aucun ne
disait quoi que ce soit sur le **fonctionnement**. Trois choses restent entièrement non prouvées :

1. Flux n'a jamais réconcilié le dépôt.
2. Le montage CSI des secrets depuis Key Vault n'a jamais été exécuté.
3. La base Tutor n'a jamais démarré sur Kubernetes.

Ces trois points sont l'objet réel de la qualification AKS. Le reste est du contrôle de conformité.
