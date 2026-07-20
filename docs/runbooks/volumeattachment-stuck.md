# Pod : "Multi-Attach error" après un force-delete

## Symptôme

Après un `kubectl delete pod --grace-period=0 --force` (ex: sur un pod Vault
bloqué), le nouveau pod recréé au même endroit (ou ailleurs) reste en `Pending`
avec :

```
Multi-Attach error for volume "pvc-..." Volume is already exclusively attached to one node and can't be attached to another
```

## Cause

Le force-delete du pod ne détache pas proprement le volume. L'objet
`VolumeAttachment` sous-jacent continue de pointer vers l'ancien node, même si
plus aucun pod n'y référence ce volume.

```bash
kubectl get volumeattachment | grep <pvc-uid-ou-nom>
```

## Fix

**Vérifier d'abord** que l'ancien node est bien sain (`Ready`) et qu'aucun pod
dessus ne référence encore réellement ce volume — un `VolumeAttachment`
légitimement actif ne doit jamais être supprimé de force.

```bash
kubectl get node <old-node> 
kubectl get pods -A --field-selector spec.nodeName=<old-node> -o wide

# une fois confirmé qu'aucun pod vivant n'a besoin de ce volume
kubectl delete volumeattachment <name>
```

Le nouveau pod peut alors monter le volume normalement.

## Prévention

Éviter `--grace-period=0 --force` sur des pods StatefulSet avec volumes
attachés sauf en dernier recours (node mort/inaccessible) ; préférer laisser
le contrôleur K8s effectuer un shutdown propre, ou drainer le node
explicitement (`kubectl drain`) qui gère le détachement correctement.
