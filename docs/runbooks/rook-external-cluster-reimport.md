# Secrets Rook CSI (`rook-csi-*`, `rook-ceph-mon`) disparus/désynchronisés

## Symptôme

Un ou plusieurs des Secrets suivants, dans le namespace du cluster Ceph
externe, sont absents ou contiennent des credentials périmées :

```bash
kubectl -n rook-ceph-external get secret \
  rook-csi-rbd-node rook-csi-rbd-provisioner \
  rook-csi-cephfs-node rook-csi-cephfs-provisioner \
  rook-ceph-mon rgw-admin-ops-user
```

Symptômes typiques en aval : CSI provisioner en `CrashLoopBackOff`,
`CephCluster` qui ne devient jamais `Connected`, `ConfigMap` `rook-ceph-mon-endpoints`
vide ou périmée.

## Cause

Ces objets sont générés par le script d'import externe Rook, pas gérés par
Helm/Kustomize/GitOps — ils peuvent être accidentellement supprimés (purge
namespace, cascade ArgoCD, nettoyage manuel trop large) sans que rien ne les
recrée automatiquement.

## Fix

Re-générer les objets en relançant la paire de scripts Rook, **avec la version
qui matche exactement le chart Rook déployé** :

```bash
# 1. Récupérer le script producteur (côté cluster Ceph, ex: hôte Proxmox)
#    Toujours le refetch, ne pas réutiliser une copie locale périmée.
ssh <ceph-admin-host> 'cat /root/rook/create-external-cluster-resources.py' \
  > create-external-cluster-resources.py

# 2. Récupérer le script d'import, à la version EXACTE du rook-ceph-operator
#    déployé (vérifier avec `kubectl -n rook-ceph get deploy rook-ceph-operator -o jsonpath='{.spec.template.spec.containers[0].image}'`)
curl -sL https://raw.githubusercontent.com/rook/rook/v1.20.1/deploy/examples/import-external-cluster.sh \
  -o import-external-cluster.sh

# 3. Exécuter côté cluster Ceph pour produire le JSON de credentials
python3 create-external-cluster-resources.py --rbd-data-pool-name <pool> --format bash > /tmp/rook-external-creds.sh

# 4. Importer côté K8s (variables d'env issues de l'étape 3)
source /tmp/rook-external-creds.sh
bash import-external-cluster.sh
```

Vérifier ensuite que le `ConfigMap rook-ceph-mon-endpoints` et tous les
Secrets listés ci-dessus sont repeuplés.

## Prévention

- Ne jamais faire de `kubectl delete` large (`--all`, wildcard de label) dans
  le namespace `rook-ceph-external` sans lister d'abord précisément ce qui va
  être supprimé.
- Après toute ré-importation, voir aussi
  [csi-nodeplugin-stale-config.md](csi-nodeplugin-stale-config.md) — les pods
  CSI déjà démarrés ne relisent pas automatiquement la nouvelle config.
