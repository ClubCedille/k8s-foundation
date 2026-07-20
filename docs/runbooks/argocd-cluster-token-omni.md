# ArgoCD → cluster spoke géré par Omni : `Unauthorized` / JWT invalide

## Symptôme

Le hub ArgoCD (`k8s-shared`) ne peut pas se connecter au cluster spoke exposé
par un Omni self-hosted (`omni.etsmtl.club:8100`) :

```bash
argocd cluster get k8s-poc
# ou
kubectl --context cedille-k8s-shared -n argocd get application k8s-poc-cilium -o yaml
# condition: "Unauthorized" ou timeouts
```

Si le token a été généré manuellement (Secret `kubernetes.io/service-account-token`
créé à la main), l'erreur côté Omni est typiquement :

```
token is unverifiable: error while executing keyfunc: key not found, ID "..."
```

... même si le endpoint JWKS du cluster contient bel et bien la clé
correspondante (`/openid/v1/jwks`).

## Cause

Omni fait sa propre validation JWT/JWKS des tokens de service account, et un
token créé "à la main" côté Kubernetes (Secret `type:
kubernetes.io/service-account-token` classique) ne passe pas cette validation
de façon fiable, même après régénération du token ou redémarrage du container
Omni. La cause exacte n'a pas été identifiée (bug/limitation Omni), mais le
contournement fonctionne à 100%.

## Fix

Ne PAS créer le token manuellement. Utiliser le mécanisme d'émission propre
d'Omni :

```bash
omnictl kubeconfig ./argocd-manager.kubeconfig \
  --service-account \
  --user=argocd-manager \
  --cluster=k8s-poc

# valider AVANT de le pousser dans ArgoCD
kubectl --kubeconfig=./argocd-manager.kubeconfig get nodes

# extraire le bearer token du kubeconfig généré et le mettre dans le Secret
# de cluster ArgoCD (namespace argocd, label argocd.argoproj.io/secret-type=cluster)
kubectl --context cedille-k8s-shared -n argocd get secret cluster-omni-etsmtl-club-k8s-poc -o json \
  | jq -r '.data.config' | base64 -d
# -> remplacer la valeur bearerToken par le nouveau token, puis:
kubectl --context cedille-k8s-shared -n argocd patch secret cluster-omni-etsmtl-club-k8s-poc \
  --type merge -p "{\"data\":{\"config\":\"$(echo -n "$NEW_CONFIG_JSON" | base64 -w0)\"}}"
```

Toujours valider le token directement avec `kubectl get nodes` en isolant le
kubeconfig généré, **avant** de le câbler dans le Secret ArgoCD — ça isole un
bug de token d'un bug de câblage Secret/ArgoCD.

## Prévention

Pour tout nouveau cluster géré par Omni ajouté au hub ArgoCD, utiliser
systématiquement `omnictl kubeconfig --service-account --user=<name>
--cluster=<name>` dès le départ plutôt que de créer un ServiceAccount +
Secret token à la main.
