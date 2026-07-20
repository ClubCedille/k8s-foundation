# CSI nodeplugin monte des volumes en échec après ré-import Ceph

## Symptôme

Après une ré-importation de cluster Ceph externe (voir
[rook-external-cluster-reimport.md](rook-external-cluster-reimport.md)), les
Secrets/ConfigMaps sont correctement repeuplés, le `CephCluster` passe
`Connected`, mais des pods applicatifs restent bloqués en
`ContainerCreating`/`Init` avec des erreurs de montage CephFS/RBD.

```bash
kubectl -n rook-ceph-external exec -it <cephfs-nodeplugin-pod> -- cat /etc/ceph-csi-config/config.json
# {} ou config vide/périmée
```

## Cause

Les pods CSI nodeplugin (DaemonSet `rook-ceph.cephfs.csi.ceph.com-nodeplugin`,
`rook-ceph.rbd.csi.ceph.com-nodeplugin`) qui ont démarré **avant** que le
`CephCluster` ne se connecte avec succès ont mis en cache un fichier de config
vide monté depuis un ConfigMap qui, à ce moment-là, n'avait pas encore été
peuplé. Ils ne relisent pas ce fichier après coup.

## Fix

Forcer le redémarrage des pods nodeplugin pour qu'ils remontent le ConfigMap
maintenant peuplé :

```bash
kubectl -n rook-ceph-external delete pods -l app=rook-ceph.cephfs.csi.ceph.com-nodeplugin
kubectl -n rook-ceph-external delete pods -l app=rook-ceph.rbd.csi.ceph.com-nodeplugin
```

Revérifier le contenu du config après redémarrage :

```bash
kubectl -n rook-ceph-external exec -it <pod> -- cat /etc/ceph-csi-config/config.json
```

## Prévention

Après toute ré-importation de credentials Ceph externe, vérifier l'ordre :
attendre que `CephCluster` soit `Connected` **avant** de considérer les
nodeplugins comme fonctionnels ; si des pods CSI existaient déjà avant
l'import, les redémarrer systématiquement par précaution plutôt que d'attendre
un échec de montage pour le découvrir.
