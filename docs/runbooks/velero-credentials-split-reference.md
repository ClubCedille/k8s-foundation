# Velero : `FailedMount` secret introuvable malgré BSL correcte

## Symptôme

Le `BackupStorageLocation` Velero référence le bon Secret, mais les pods
Velero/node-agent restent bloqués en `ContainerCreating` avec :

```
FailedMount ... secret "velero-b2-credentials" not found
```

... alors que le Secret réellement présent dans le cluster s'appelle
différemment (ex: `velero-s3-credentials`).

## Cause

Le chart Helm Velero a **deux références de secret indépendantes** qu'il faut
aligner manuellement :

1. `credentials.existingSecret` (values.yaml) — contrôle le volume monté par
   le Deployment/DaemonSet.
2. `BackupStorageLocation.spec.credential.name` — référencé séparément dans
   les ressources custom du repo (`patch.yaml` par overlay).

Le values.yaml partagé (`common/velero/helm/values.yaml`) hardcode
`velero-b2-credentials` (Backblaze B2), le backend par défaut. Un overlay de
cluster utilisant un autre backend (ex: GarageHQ pour k8s-poc) doit surcharger
**les deux**, pas seulement la BSL.

## Fix

Dans le `kustomization.yaml` de l'overlay du cluster concerné, ajouter un
`valuesInline` qui aligne `credentials.existingSecret` sur le même nom de
Secret que celui déjà utilisé dans `patch.yaml` pour la BSL :

```yaml
# common/velero/overlays/k8s-poc/kustomization.yaml
helmCharts:
  - name: velero
    valuesFile: "../../helm/values.yaml"
    valuesInline:
      credentials:
        existingSecret: velero-s3-credentials   # doit matcher patch.yaml
patches:
  - path: patch.yaml   # contient spec.credential.name: velero-s3-credentials
```

Vérifier après sync que les deux références concordent et que le Secret cité
existe bien dans le namespace `velero` :

```bash
kubectl -n velero get backupstoragelocation -o jsonpath='{.items[0].spec.credential}'
kubectl -n velero get secret velero-s3-credentials
```

## Prévention

Pour tout nouveau cluster utilisant un backend de stockage différent du
défaut partagé, grep les deux occurrences avant de déployer :

```bash
grep -rn "existingSecret\|credential:" common/velero/
```
