# Nouveau cluster : collision de noms ApplicationSet entre repos

## Symptôme

Deux repos gérés par le même hub ArgoCD (`k8s-base` pour les clusters
existants, `k8s-foundation` pour un nouveau cluster) définissent des
`ApplicationSet` avec le **même nom** (ex: `vault`, `velero`, `cert-manager`)
dans le même namespace `argocd`. Le second appliqué "gagne" silencieusement
la propriété de l'objet, et son generator `clusters: {}` (sans sélecteur)
matche **tous** les clusters déjà enregistrés dans le hub — pas seulement
celui visé.

Conséquence observée en incident réel : le générateur sans filtre d'un
ApplicationSet `gitops-argoapps` a fini par supprimer une `Application` gérant
ArgoCD lui-même sur un cluster qui n'était pas la cible voulue, causant une
cascade de suppression de tout ArgoCD.

## Cause

- Absence de garde-fou de nommage : deux ApplicationSets homonymes dans le
  même namespace se marchent dessus au lieu d'être rejetés.
- Absence de garde-fou de portée : `clusters: {}` cible tous les clusters
  enregistrés dans le hub, par design, sauf si un `selector` restreint la
  cible.

## Fix / pattern à appliquer sur TOUT nouveau repo/stack ajouté au hub

**1. Préfixer les ApplicationSets du nouveau repo** pour éviter toute
collision de nom avec les stacks existantes :

```yaml
metadata:
  name: kf-vault   # au lieu de "vault", qui existe déjà dans k8s-base
```

**2. Labelliser chaque cluster dans le Secret ArgoCD** avec la stack à
laquelle il appartient :

```bash
kubectl -n argocd label secret cluster-omni-etsmtl-club-k8s-poc stack=k8s-foundation
kubectl -n argocd label secret <secret-k8s-shared> stack=k8s-base
```

**3. Ajouter un `selector` sur le generator `clusters` de CHAQUE
ApplicationSet**, dans les deux repos :

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          stack: k8s-foundation   # ou k8s-base, selon le repo
```

## Vérification

Avant de merger/appliquer un nouvel ApplicationSet :

```bash
# aucune collision de nom entre les deux repos
comm -12 <(yq '.metadata.name' k8s-base/**/*.argoapp.yaml | sort) \
         <(yq '.metadata.name' k8s-foundation/**/*.argoapp.yaml | sort)

# tout generator clusters a un selector (grep sanity check)
grep -L "selector:" **/*.argoapp.yaml   # doit être vide
```

## Prévention

Ce pattern (préfixe de nom + label de cluster + selector obligatoire) doit
être appliqué **avant** le premier `kubectl apply` du bootstrap d'un nouveau
repo/stack sur le hub, jamais après coup — c'est un garde-fou préventif, pas
un fix a posteriori.
