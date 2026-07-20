# Ressource K8s bloquée en `Terminating` indéfiniment

## Symptôme

Une ressource (namespace, `Application` ArgoCD, CR Rook comme `CephCluster`/
`CephObjectStore`, `ConfigMap`, etc.) a un `metadata.deletionTimestamp` défini
mais ne disparaît jamais, même après plusieurs minutes.

```bash
kubectl get application k8s-poc-rook -n argocd
# STATUS montre toujours l'objet, avec un âge qui grandit
kubectl get application k8s-poc-rook -n argocd -o jsonpath='{.metadata.deletionTimestamp}{"\n"}{.metadata.finalizers}'
```

## Cause

Le controller propriétaire du finalizer était mort (ou n'a jamais tourné) au
moment où la suppression a été initiée. Le finalizer reste posé pour toujours
tant que ce controller (ou un humain) ne le retire pas :

- `resources-finalizer.argocd.argoproj.io` (ArgoCD `Application`)
- `csi.ceph.com/cleanup` (PVC/PV)
- `ceph.rook.io/disaster-protection` (`CephCluster`, `CephObjectStore` — voir
  aussi la note ci-dessous)
- `cephcluster.ceph.rook.io` (namespace contenant un CephCluster)

C'est le pattern root-cause le plus fréquent rencontré pendant l'incident de
cascade ArgoCD (contrôleur ArgoCD tué en plein milieu d'une purge en cascade)
et pendant les migrations Rook (operator redémarré/remplacé pendant qu'un CR
était en cours de suppression).

## Fix

**Étape 1 — vérifier qu'il n'y a plus de consommateur vivant** de la ressource
(aucun pod ne monte le volume, aucun autre objet ne la référence). Ne JAMAIS
sauter cette étape.

**Étape 2 — retirer les finalizers directement**, sans relancer la logique de
purge du controller (qui pourrait re-déclencher une cascade si le controller
vient tout juste de redevenir vivant) :

```bash
kubectl patch application k8s-poc-rook -n argocd \
  --type merge -p '{"metadata":{"finalizers":[]}}'

kubectl patch namespace rook-ceph-external \
  --type merge -p '{"metadata":{"finalizers":[]}}'

kubectl patch cephcluster my-cluster -n rook-ceph-external \
  --type merge -p '{"metadata":{"finalizers":[]}}'
```

L'objet disparaît immédiatement après le patch.

## Cas particulier : `ceph.rook.io/disaster-protection`

Ce finalizer est une protection anti-suppression-accidentelle de Rook sur
`CephCluster`/`CephObjectStore`. Il est sécuritaire de le retirer **quand le
cluster Ceph est externe** (`external: true` — les données Ceph vivent hors du
cluster K8s, le CR n'est qu'une référence). **Ne pas appliquer ce raisonnement
à un cluster Ceph possédé (non-external) par Rook** : dans ce cas le finalizer
protège réellement contre une perte de données, et il faut d'abord confirmer
qu'aucune destruction de données réelle n'est en jeu avant de le retirer.

## Prévention

Avant toute action destructive en masse sur ArgoCD (désactivation de sync
auto, suppression d'ApplicationSet), désactiver l'`automated sync` de
l'Application parente pour éviter qu'un controller vivant ne relance une
purge en cascade pendant la remédiation :

```bash
kubectl patch application <app> -n argocd --type json \
  -p '[{"op":"remove","path":"/spec/syncPolicy/automated"}]'
```
