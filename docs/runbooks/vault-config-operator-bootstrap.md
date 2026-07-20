# `vault-config-operator` : login 403/400 en boucle, ArgoCD Degraded sur toute app dépendant de Vault

## Symptôme

Une `Application` ArgoCD qui dépend d'un secret géré via `VaultStaticSecret`
(ex: `k8s-poc-rook`, `k8s-poc-storage-config`) reste `Degraded`, alors
qu'aucune ressource individuelle listée par ArgoCD ne montre explicitement un
statut malsain. En creusant la `VaultStaticSecret` elle-même :

```bash
kubectl -n <ns> get vaultstaticsecret <name> -o jsonpath='{.status.conditions}'
```

```
Failed to sync the secret ... err=Error making API request.
URL: PUT http://vault.<ns>.svc.cluster.local:8200/v1/auth/kubernetes/login
Code: 403. Errors:
* permission denied
```

Et en creusant les CRs `redhatcop.redhat.io` du `vault-config-operator`
lui-même (`Policy`, `KubernetesAuthEngineRole`, `SecretEngineMount`) dans le
namespace `vault` :

```bash
kubectl -n vault get policies.redhatcop.redhat.io,kubernetesauthengineroles,secretenginemounts \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[0].status,MSG:.status.conditions[0].message
```

```
config-admin   False   Put ".../auth/kubernetes/login": dial tcp: lookup vault.vault.svc.cluster.local: no such host
# ou, sur un Vault qui a déjà fonctionné auparavant :
config-admin   False   Put ".../auth/kubernetes/login" Code: 400. Errors: * invalid role name "config-admin"
```

## Cause n°1 — bootstrap `auth/kubernetes` jamais fait (nouveau cluster)

`vault-config-operator` gère déclarativement policies/rôles/mounts Vault via
des CRs, mais **s'authentifie lui-même** contre Vault avec le rôle
`auth/kubernetes/role/config-admin` — le tout premier rôle qu'il est censé
créer. Sur un Vault fraîchement initialisé, ce rôle n'existe pas encore : rien
ne peut démarrer tant qu'un humain n'a pas fait ce bootstrap une seule fois,
directement avec le root token :

```bash
kubectl -n vault exec vault-0 -- vault auth enable kubernetes

kubectl -n vault exec vault-0 -- sh -c \
  'vault write auth/kubernetes/config \
     kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}" \
     kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token'

# policy identique à celle déclarée dans common/vault/base/resources/policies.yaml
kubectl -n vault cp config-admin-policy.hcl vault/vault-0:/tmp/config-admin-policy.hcl
kubectl -n vault exec vault-0 -- vault policy write config-admin /tmp/config-admin-policy.hcl
kubectl -n vault exec vault-0 -- rm -f /tmp/config-admin-policy.hcl

kubectl -n vault exec vault-0 -- vault write auth/kubernetes/role/config-admin \
  bound_service_account_names="vault,default,controller-manager" \
  bound_service_account_namespaces="vault" \
  policies="config-admin" \
  ttl=24h
```

Après ce bootstrap, `vault-config-operator` s'authentifie normalement et
reconcilie tout le reste (mounts, policies, rôles) tout seul à partir des CRs
déjà en git.

**Action à fort impact** : cette policy accorde `sudo` sur `sys/*`,
`auth/*`, `identity/*` — traiter comme n'importe quel autre changement de
policy Vault root-équivalent, confirmation explicite requise avant
application sur un cluster partagé.

## Cause n°2 — le même bootstrap peut disparaître sur un cluster déjà en prod

Constaté sur `k8s-shared` : le rôle `config-admin` avait fonctionné avec succès
jusqu'au 2026-06-25 (`SecretEngineMount/github` avec condition
`ReconcileSuccessful` à cette date), puis s'est retrouvé absent (`invalid role
name`) après une restauration Vault faite en urgence pendant un incident
ArgoCD — alors que Vault lui-même rapportait toujours `Initialized: true`,
`Sealed: false`, avec le **même** `Cluster ID` qu'avant (donc pas une
réinitialisation complète). Le mount `kv-system` s'est retrouvé vide au même
moment.

**Hypothèse retenue** : recréer/reconfigurer l'auth Kubernetes d'un Vault
existant (par ex. en ré-appliquant un `KubernetesAuthEngineConfig` ou en
réactivant le mount `kubernetes` par erreur) peut silencieusement effacer les
rôles déjà enregistrés sous ce mount, même si le backend de stockage
(GCS/S3) sous-jacent reste intact et que Vault reste unsealed avec son
identité d'origine. Un statut `Initialized: true` / `Sealed: false` /
`Cluster ID` inchangé **ne garantit donc pas** que la configuration
`auth/kubernetes` (et les mounts KV) est intacte.

## Cause n°3 — un secret Vault référencé mais absent fait planter (pas juste échouer) l'operator

Sur `k8s-poc`, même après avoir recréé policy+rôle `config-admin`, l'operator
continuait à ne rien reconcilier. Les logs ont montré un **panic Go qui tue
tout le process** (pas juste un échec de reconcile isolé) :

```
Observed a panic in reconciler ... GitHubSecretEngineConfig ...
panic: runtime error: invalid memory address or nil pointer dereference
    .../api/v1alpha1.(*GitHubSecretEngineConfig).setInternalCredentials
```

La CR `GitHubSecretEngineConfig/gh-engine-cedille` (déployée identiquement sur
tous les clusters via `common/vault/base/resources/github-secret-engine.yaml`)
référence un secret Vault (`kv-system/github/clubcedille`, la clé privée
de la GitHub App Club Cedille) qui doit **déjà exister** avant que cette CR ne
soit reconciliée — si le chemin est vide (mount `kv-system` neuf ou vidé),
le code de l'operator (v0.8.49) ne gère pas ce cas et panique, **crashant le
pod entier** plutôt que de simplement marquer cette seule CR en échec. Toutes
les autres CRs en attente de reconciliation dans la même queue (`kv-system`,
`secret-reader`, `secret-writer`...) restent bloquées indéfiniment tant que ce
pod redémarre en boucle sur la même CR fautive.

## Fix

1. Faire le bootstrap manuel `auth/kubernetes` (Cause n°1) si absent.
2. Vérifier qu'aucune CR `redhatcop.redhat.io` référençant un secret Vault
   (`vaultSecret.path`) ne pointe vers un chemin vide avant de laisser
   l'operator tourner sans surveillance :
   ```bash
   kubectl -n vault get githubsecretengineconfig,vaultpkisecret,vaultsecret -A \
     -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec}{"\n"}{end}'
   ```
3. Si le secret référencé n'existe pas encore (cas courant sur un nouveau
   cluster, ou après une perte de données Vault comme sur k8s-shared), soit
   le pousser dans Vault avant de laisser la CR se reconcilier, soit retirer
   temporairement la CR de l'overlay du cluster concerné le temps de
   retrouver/régénérer la valeur (ex: régénérer une nouvelle clé privée côté
   GitHub App si l'ancienne est irrécupérable).

## Prévention

- Après toute restauration de Vault suite à un incident, **ne pas se fier
  uniquement à** `vault status` (`Initialized`/`Sealed`/`Cluster ID`) pour
  conclure que la configuration est intacte. Vérifier explicitement :
  ```bash
  kubectl -n vault exec vault-0 -- vault auth list
  kubectl -n vault exec vault-0 -- vault secrets list
  kubectl -n vault exec vault-0 -- vault read auth/kubernetes/role/config-admin
  ```
- Avant de déployer le socle Vault partagé (`common/vault/base/resources/`)
  sur un nouveau cluster, s'attendre à devoir bootstrapper manuellement
  `auth/kubernetes` + `config-admin` en premier (voir Cause n°1) — ce n'est
  pas automatisé et ne le sera pas tant que `vault-config-operator` a besoin
  de ce rôle pour s'authentifier lui-même.
- Traiter tout secret externe référencé par une CR `vault-config-operator`
  (clé GitHub App, credentials tierces) comme un pré-requis **bloquant et
  fragile** : son absence ne fait pas juste échouer une CR isolée, elle peut
  geler la réconciliation de tout le namespace `vault` selon la version de
  l'operator.
