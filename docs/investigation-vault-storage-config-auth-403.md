# Investigation ouverte : login Vault kubernetes-auth échoue pour le namespace `storage-config` sur k8s-poc

**Statut : cause probable identifiée le 2026-07-20 (voir "Mise à jour"), correctif préparé mais pas encore appliqué en cluster (bootstrap manuel requis).**
**Cluster concerné : `k8s-poc` (Vault interne, pod `vault-0`, namespace `vault`).**
**Impact concret : initialement `VaultStaticSecret/garagehq-credentials` (namespace `storage-config`) ; s'est révélé être une panne globale du login kubernetes-auth depuis le 2026-07-20 15:06Z (voir mise à jour).**

## Mise à jour 2026-07-20 : ce n'est pas (que) `storage-config`

En reprenant l'investigation, `vault-0` avait redémarré le jour même à
`15:06:11Z` et tourne maintenant en **Vault v1.20.0**. Constats faits en
direct sur le cluster (`omni-k8s-poc`) :

- Login manuel frais (`vault write auth/kubernetes/login`, JWT généré via
  `kubectl create token`) échoue avec le **même 403** pour `rook-ceph-external`
  aussi — namespace documenté plus bas comme fonctionnant sans problème.
- `vault-config-operator` n'a **plus reconcilié aucune CR
  `redhatcop.redhat.io` avec succès depuis `14:53:31Z`**, soit juste avant le
  redémarrage de `vault-0`. Aucun succès depuis (rôles, policies, mounts).
- Le client d'encryption global de `vault-secrets-operator` (rôle
  `secret-writer`, namespace `vault`, utilisé par le cache de TOUS les
  namespaces) échoue aussi en 403 (`Failed to create Vault client for
  storage encryption`).
- `VaultStaticSecret/rgw-admin-ops-user` (rook-ceph-external) affiche
  `Synced=True`, mais son dernier succès réel date de **la veille (2026-07-19
  11:58Z)** — pas de `spec.refreshAfter` configuré, donc pas de resync
  périodique : le statut "sain" est un résidu figé d'avant la panne, pas une
  preuve de bon fonctionnement actuel.
- Logs `vault-0` depuis le redémarrage, en boucle continue (~toutes les 4s,
  sans interruption sur 2h30+) :
  ```
  [WARN] auth.kubernetes...: A role without an audience was used to
  authenticate into Vault. Vault v1.21+ will require roles to have an
  audience.: role_name=secret-reader
  [WARN] ... role_name=secret-writer
  ```

**Hypothèse retenue** : aucun des rôles `config-admin` / `secret-writer` /
`secret-reader` (`common/vault/base/resources/roles.yaml`) ne déclare de
champ `audience`, alors que tous les clients (VSO, vault-config-operator)
envoient `audiences: ["vault"]` dans leur JWT
(`common/vault/base/helm/vault-secrets-operator.values.yaml`,
`vault-config-operator.values.yaml`). Le nouveau Vault v1.20 semble déjà
appliquer une vérification plus stricte que ce que le message d'avertissement
laisse penser, et rejette silencieusement ces logins en 403 — pour tous les
namespaces, pas seulement `storage-config` (qui n'était que le canari le plus
visible, faute de cache/état figé pour le masquer).

**Correctif préparé** : ajout de `spec.audience: vault` aux trois CR
`KubernetesAuthEngineRole` dans `common/vault/base/resources/roles.yaml`
(champ supporté par le CRD `kubernetesauthengineroles.redhatcop.redhat.io`).

**Piège de déploiement** : `vault-config-operator` s'authentifie lui-même
via le rôle `config-admin` — celui-là même qui est cassé. Une fois ce
correctif mergé et synced par ArgoCD, l'opérateur restera probablement
incapable de l'appliquer tout seul (même blocage que la "Cause n°1" du
runbook `docs/runbooks/vault-config-operator-bootstrap.md`). Il faudra
vraisemblablement un bootstrap manuel avec le root token Vault pour
réécrire `auth/kubernetes/role/config-admin` avec `bound_audiences=vault`
directement, avant que l'operator ne puisse reconcilier le reste. Non fait
faute d'accès au root token au moment de cette investigation.

## Contexte

Dans le cadre du transfert de secrets Vault (clé GitHub App, credentials RGW,
etc.) vers `k8s-poc`, on a aussi dû fournir des credentials GarageHQ
(`kv-system/default_passwords/garagehq`). En creusant pourquoi
`VaultStaticSecret/garagehq-credentials` restait en échec, deux problèmes
distincts ont été trouvés :

1. **Bug de config (corrigé)** — `common/garage/resources/garagehq-config.yaml`
   déclarait `type: kv-v2` alors que le mount `kv-system` est en réalité
   KV v1 (même mount que `rgw-admin-ops-user`, qui utilise correctement
   `type: kv-v1`). Corrigé dans ce repo.
2. **Bug non résolu (celui documenté ici)** — même après avoir poussé les
   credentials dans Vault et corrigé le type, `vault-secrets-operator` (VSO)
   n'arrive JAMAIS à s'authentifier auprès de Vault pour le namespace K8s
   `storage-config` spécifiquement, alors que la même configuration
   (rôle `secret-reader`, méthode kubernetes, policies identiques)
   fonctionne parfaitement pour d'autres namespaces (`rook-ceph-external`,
   `capra-outlinewiki`, `vault`).

## Symptôme observé

```bash
kubectl -n storage-config get vaultstaticsecret garagehq-credentials
# SYNCED=False HEALTHY=False READY=False

kubectl -n storage-config get vaultstaticsecret garagehq-credentials -o jsonpath='{.status.conditions}'
```

```
Failed to sync the secret ... err=Error making API request.
URL: PUT http://vault.vault.svc.cluster.local:8200/v1/auth/kubernetes/login
Code: 403. Errors:
* permission denied
```

Logs de `vault-secrets-operator-controller-manager` (namespace `vault`) :

```
"msg":"Failed to get NewClientWithLogin", ... "error":"... Code: 403 ... permission denied"
"msg":"Failed to restore client from storage", "logger":"cachingClientFactory", ...
"msg":"Failed to create Vault client for storage encryption", "logger":"setEncryptionClient", ...
```

Point important : ce **n'est pas** une erreur de policy Vault classique
(refus d'accès à un chemin après authentification) — c'est le **login
kubernetes-auth lui-même** qui échoue avec 403, avant toute évaluation de
policy sur un chemin de secret.

## Ce qui a été vérifié et exclu (dans l'ordre)

Toutes ces vérifications ont été faites directement contre Vault sur
`k8s-poc` (`kubectl exec -n vault vault-0 -- vault ...` avec le root token),
et en reproduisant manuellement le login avec un JWT frais généré via
`kubectl -n storage-config create token <sa> --audience=vault --duration=10m`.

1. **Policy/role mal configurés ?** Non — `auth/kubernetes/role/secret-reader`
   a `bound_service_account_names=[*]` et `bound_service_account_namespaces=[*]`,
   donc n'importe quel SA de n'importe quel namespace devrait pouvoir
   s'authentifier. La policy `secret-reader` elle-même autorise bien
   `kv-system/default_passwords/*` en lecture.

2. **Régression globale du kubernetes-auth (p.ex. après un restart de
   `vault-0`) ?** Non — `VaultStaticSecret/rgw-admin-ops-user`
   (namespace `rook-ceph-external`, même rôle `secret-reader`) continue à se
   synchroniser sans problème pendant toute la fenêtre de test.

3. **Identity entity Vault corrompue pour ce SA ?** Non — l'entity
   existante (`entity_f9f7ea58`, créée le 2026-07-06, liée au SA
   `storage-config/default`) a été supprimée
   (`vault delete identity/entity/name/entity_f9f7ea58`) pour forcer Vault à
   en recréer une neuve au prochain login. Le login échoue toujours
   identiquement après suppression.

4. **Problème spécifique au service account `default` de ce namespace ?**
   Non — un nouveau SA jetable (`vault-test-sa`) créé dans `storage-config`
   spécifiquement pour le test échoue exactement pareil. Le problème n'est
   donc pas lié à un SA en particulier, mais bien au **namespace
   `storage-config` lui-même**.

5. **Rejet côté API server Kubernetes (TokenReview) ?** Non — en soumettant
   manuellement le même JWT à l'API Kubernetes via un objet `TokenReview`
   direct (`kind: TokenReview`, `spec.token=<jwt>`, `spec.audiences=[vault]`),
   l'API server valide le token sans problème
   (`authenticated: true`, `username: system:serviceaccount:storage-config:default`).
   Donc ce n'est pas Kubernetes qui refuse le token — le refus vient
   spécifiquement de l'intérieur du plugin kubernetes-auth de Vault.

6. **Cache client VSO corrompu pour ce namespace ?** Non — aucun secret de
   cache VSO n'existe dans `storage-config` (`kubectl -n storage-config get
   secrets` → vide). Le seul secret de cache global est
   `vso-cc-storage-hmac-key` dans le namespace `vault`, partagé par toutes
   les apps.

7. **Namespaces Vault (feature Enterprise) ou multi-mounts kubernetes-auth ?**
   Non — `vault namespace list` → aucun namespace Vault configuré (root
   uniquement). `vault auth list` → un seul mount kubernetes
   (`auth_kubernetes_26e36b22`).

8. **Policy globale (`common`/`default`) contenant une règle deny visant
   `storage-config` ?** Non — les deux policies ont été lues intégralement,
   aucune mention de `storage-config` ni de règle deny.

9. **Audit log Vault** (device `file` activé temporairement puis désactivé
   après investigation) : montre `"auth":{"policy_results":{"allowed":true}}`
   ET `"error":"permission denied"` sur la même entrée — confirmant que le
   refus vient de la logique interne du plugin kubernetes-auth (probablement
   liée à la validation du JWT / TokenReview côté Vault), pas du moteur de
   policy ACL de Vault.

## Hypothèse restante (non vérifiée)

Le refus semble venir de l'appel TokenReview effectué **par Vault
lui-même** (via son propre `token_reviewer_jwt`) vers l'API Kubernetes,
d'une manière différente de l'appel manuel qui, lui, réussit. Pistes non
explorées faute d'accès :

- Logs Vault en niveau `trace` (le niveau actuel est `debug`, insuffisant
  pour voir le détail exact de l'appel TokenReview émis par le plugin
  kubernetes-auth).
- Vérifier si le nom du namespace `storage-config` interagit mal avec un
  cache interne du plugin (nom généré dynamiquement, collision de clé de
  cache, etc.) — semble spécifique au *nom* du namespace K8s, pas au SA.
- Comparer avec un test où l'on renomme temporairement (ou clone) le
  namespace `storage-config` sous un autre nom pour voir si le problème
  suit le nom ou la ressource.
- Vérifier la configuration `disable_iss_validation`/`issuer` de
  `auth/kubernetes/config` de plus près (actuellement
  `disable_iss_validation: true`, `issuer: n/a`) — semble correct mais reste
  la seule zone de config non testée isolément par élimination.

## État actuel du cluster (à la fin de l'investigation)

- `common/garage/resources/garagehq-config.yaml` : `type: kv-v1` (corrigé,
  commit à faire/vérifier dans ce repo).
- `kv-system/default_passwords/garagehq` : peuplé sur **k8s-poc** et
  **k8s-shared** (clé Garage `garagehq-credentials`,
  `GKde6595c456c1626f1264eaac`, générée via
  `ssh localadmin@10.0.21.50 garage key create garagehq-credentials`).
- `VaultStaticSecret/garagehq-credentials` (namespace `storage-config`,
  k8s-poc) : toujours en échec, bloqué par ce problème d'authentification.
- Aucune ressource Vault de diagnostic laissée en place (audit device
  désactivé, entity/SA de test supprimés, tokens temporaires révoqués/expirés).

## Prochaine étape suggérée

Passer Vault en `log_level = "trace"` temporairement (via le ConfigMap
`vault-config` / `common/vault/base/helm/vault.values.yaml`,
`server.standalone.config`), reproduire le login échoué pour
`storage-config`, puis comparer trace-par-trace avec un login réussi
(`rook-ceph-external`) pour repérer où le chemin diverge à l'intérieur du
plugin kubernetes-auth.
