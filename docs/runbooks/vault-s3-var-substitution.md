# Vault : credentials S3 jamais substituées dans le HCL

## Symptôme

Le pod Vault (image `ghcr.io/clubcedille/vault:*`) configuré avec un backend
`storage "s3"` échoue à démarrer ou à s'authentifier contre le stockage S3
(RGW, GCS, B2...), et en inspectant la config effective on trouve la variable
non substituée, littéralement :

```bash
kubectl -n vault exec vault-0 -- cat /vault/config/extraconfig-from-values.hcl | grep -A2 'storage "s3"'
# access_key = "${RGW_ACCESS_KEY}"   <-- jamais remplacé, envoyé tel quel à RGW
```

## Cause

Le script de démarrage custom de cette image (`/bin/sh -ec ... cp ...; sed -Ei
...; docker-entrypoint.sh vault server ...`) fait uniquement une substitution
`sed` fixe sur **6 tokens précis** : `HOST_IP`, `POD_IP`, `HOSTNAME`,
`API_ADDR`, `TRANSIT_ADDR`, `RAFT_ADDR`. Ce n'est **pas** un `envsubst`
générique, et ce n'est **pas** le templating `{{ env "VAR" }}` natif de Vault.
Tout autre `${VAR}` dans le HCL de config est envoyé tel quel, littéralement,
au backend de stockage.

Vérifiable directement en inspectant les args du container :

```bash
kubectl -n vault get pod vault-0 -o jsonpath='{.spec.containers[0].args}'
```

## Fix

Ne PAS mettre `access_key`/`secret_key` dans le bloc `storage "s3"` du HCL.
Les omettre complètement, et fournir les credentials comme variables d'env
standard AWS sur le container — le SDK AWS utilisé par le backend S3 de Vault
les détecte automatiquement (default credential chain) :

```yaml
storage "s3" {
  endpoint            = "http://10.0.21.59:7480"
  bucket              = "vault-k8s-poc"
  region              = "default"
  s3_force_path_style = true
  disable_ssl         = true
  # access_key/secret_key volontairement absents, voir env AWS_* ci-dessous
}
```

```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef: {name: rgw-vault-k8s-poc, key: access_key}
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef: {name: rgw-vault-k8s-poc, key: secret_key}
```

Avant de câbler les credentials dans le pod, valider qu'elles sont correctes
en testant un appel S3 signé directement contre l'endpoint (isole un bug de
credential d'un bug de templating) :

```bash
python3 sign_and_put.py --endpoint http://10.0.21.59:7480 --bucket vault-k8s-poc \
  --access-key "$AK" --secret-key "$SK"
```

## Prévention

Sur tout nouveau cluster utilisant cette image Vault custom avec un backend
S3 non-GCS, ne jamais mettre de `${VAR}` arbitraire dans le HCL en dehors des
6 tokens supportés — passer systématiquement par des env vars `AWS_*` /
`GOOGLE_APPLICATION_CREDENTIALS`-style consommées par le SDK sous-jacent.
