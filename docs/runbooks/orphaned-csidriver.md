# `ceph-csi-operator` : "invalid driver name" / CSIDriver déjà utilisé

## Symptôme

Après avoir déployé/migré vers le modèle `ceph-csi-operator` (charts Rook
≥ v1.13, sub-chart `ceph-csi-operator`), le controller-manager boucle en
erreur de réconciliation :

```bash
kubectl -n rook-ceph logs deploy/ceph-csi-op-controller-manager | grep -i "invalid driver name"
# "invalid driver name": Desired name already in use by a different CSI Driver
```

Les objets `Driver`/`OperatorConfig` (CRDs `csi.ceph.io`) restent en erreur, et
les pods CSI (`rook-ceph.rbd.csi.ceph.com`, `rook-ceph.cephfs.csi.ceph.com`) ne
se créent jamais.

Rencontré 3 fois cette session (sandbox, puis 2 fois sur k8s-poc après la
migration Rook v1.10.13 → v1.20.1).

## Cause

Des objets `CSIDriver` (cluster-scoped, `storage.k8s.io/v1`) issus d'une
installation Rook antérieure (legacy, ou une toute autre installation) existent
déjà sous le même nom (`rook-ceph.rbd.csi.ceph.com`, etc.) **sans** l'annotation
`csi.ceph.io/ownerref` que pose le nouveau `ceph-csi-operator`. Le controller
refuse de "voler" un objet qu'il ne reconnaît pas comme sien.

```bash
kubectl get csidriver rook-ceph.rbd.csi.ceph.com -o jsonpath='{.metadata.annotations}'
# {} ou absent de "csi.ceph.io/ownerref"
```

## Fix

Supprimer l'objet orphelin : l'operator le recrée immédiatement, cette fois
avec l'annotation correcte.

```bash
kubectl delete csidriver rook-ceph.rbd.csi.ceph.com
kubectl delete csidriver rook-ceph.cephfs.csi.ceph.com

# Vérifier la recréation avec l'ownerref
kubectl get csidriver rook-ceph.rbd.csi.ceph.com -o jsonpath='{.metadata.annotations}'
```

## Prévention

Lors d'une migration Rook legacy → `ceph-csi-operator` sur un cluster qui a
déjà eu une installation Ceph/CSI antérieure (même partielle ou abandonnée),
vérifier systématiquement les `CSIDriver` existants **avant** de déployer le
nouveau chart :

```bash
kubectl get csidriver
```

Si un driver du même nom existe déjà sans le bon ownerref, le supprimer en
amont évite le cycle d'erreur/attente/investigation.
