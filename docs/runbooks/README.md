# Runbooks — dictionnaire symptôme → fix

Ce dossier documente les procédures rencontrées et répétées lors des déploiements
de nouveaux clusters dans le modèle hub-and-spoke ArgoCD (`k8s-shared` = hub,
`k8s-base`/`k8s-foundation` = repos de manifests). Chaque fiche part d'un
symptôme observable (erreur, statut ArgoCD, comportement de pod) et donne la
commande de diagnostic + le fix, avec un exemple concret tiré de l'onboarding
de `k8s-poc`.

À consulter en premier lors de l'ajout d'un nouveau cluster au hub, ou quand un
cluster existant réapparaît dans un état bizarre après une ré-importation Ceph,
un redémarrage de node, ou un incident ArgoCD.

## Index

| Symptôme | Fiche |
|---|---|
| Ressource K8s bloquée en `Terminating` indéfiniment | [stuck-terminating-finalizers.md](stuck-terminating-finalizers.md) |
| `ceph-csi-operator` : "invalid driver name" / CSIDriver déjà utilisé | [orphaned-csidriver.md](orphaned-csidriver.md) |
| ArgoCD → cluster spoke : `Unauthorized` / token JWT invalide (Omni) | [argocd-cluster-token-omni.md](argocd-cluster-token-omni.md) |
| Secrets Rook CSI (`rook-csi-*`, `rook-ceph-mon`) disparus/désynchronisés | [rook-external-cluster-reimport.md](rook-external-cluster-reimport.md) |
| CSI nodeplugin monte des volumes en échec après ré-import Ceph | [csi-nodeplugin-stale-config.md](csi-nodeplugin-stale-config.md) |
| Pod Vault : "Multi-Attach error" après un force-delete | [volumeattachment-stuck.md](volumeattachment-stuck.md) |
| Vault : credentials S3 jamais substituées dans le HCL | [vault-s3-var-substitution.md](vault-s3-var-substitution.md) |
| Velero : `FailedMount` secret introuvable malgré BSL correcte | [velero-credentials-split-reference.md](velero-credentials-split-reference.md) |
| Nouveau cluster : collision de noms ApplicationSet entre repos | [applicationset-cluster-selector-guardrail.md](applicationset-cluster-selector-guardrail.md) |
| ArgoCD selfHeal annule un correctif appliqué à la main | [argocd-selfheal-reverts-manual-changes.md](argocd-selfheal-reverts-manual-changes.md) |
| Pod (hubble-relay/hubble-ui...) : DNS/réseau cassés malgré un DaemonSet Cilium sain | [cilium-pod-missing-endpoint.md](cilium-pod-missing-endpoint.md) |
