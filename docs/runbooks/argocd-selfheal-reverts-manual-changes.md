# ArgoCD selfHeal annule un correctif appliqué à la main

## Symptôme

Un `kubectl apply`/`kubectl patch` manuel sur une ressource gérée par une
`Application` ArgoCD avec `syncPolicy.automated.selfHeal: true` semble
fonctionner, mais disparaît quelques minutes plus tard sans intervention
apparente — la ressource revient exactement à l'état du dernier commit `git`
suivi par l'Application.

Rencontré en testant une version de chart alternative (`coroot-operator`
v0.8.2) directement via `kubectl apply`, en bypassant git : l'Application
`common` (k8s-base), qui suit toujours `master` (donc v0.9.7), a reverté le
changement en quelques minutes.

## Cause

`selfHeal: true` fait qu'ArgoCD reconcilie en continu l'état live vers l'état
déclaré en git, y compris pour annuler des drifts introduits manuellement —
c'est le comportement voulu du GitOps, mais il piège les tests/hotfix "live
only" qui ne passent pas par git.

## Fix / façon de travailler

- **Tout changement destiné à persister doit passer par git** : branche → PR →
  merge → laisser ArgoCD synchroniser, jamais de `kubectl apply` direct comme
  fix définitif sur une ressource `selfHeal`.
- **Pour tester une valeur avant de committer**, désactiver temporairement
  `automated` sur l'Application concernée, tester, puis soit committer le
  résultat validé, soit réactiver `automated` pour repartir sur l'état git :

```bash
kubectl patch application <app> -n argocd --type json \
  -p '[{"op":"remove","path":"/spec/syncPolicy/automated"}]'

# ... tester le changement manuel ...

kubectl patch application <app> -n argocd --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":true,"prune":true}}}}'
```

- **Pour un fix réellement temporaire en attendant un merge** (ex: neutraliser
  une règle `ip rule` erronée sur des pods déjà en cours d'exécution, en
  attendant qu'une PR corrigeant le manifeste soit mergée), c'est acceptable
  **à condition d'être conscient** que : (a) selfHeal ne touche pas l'intérieur
  d'un pod déjà démarré (donc ce type de patch runtime survit, contrairement à
  un patch sur un objet K8s déclaratif), mais (b) tout restart/recreate du pod
  perd le fix tant que la PR n'est pas mergée.

## Prévention

Avant de tester une hypothèse via `kubectl apply`/`patch` sur une ressource
gérée par ArgoCD, vérifier d'abord si `selfHeal` est actif :

```bash
kubectl get application <app> -n argocd -o jsonpath='{.spec.syncPolicy.automated}'
```

Si oui, désactiver `automated` avant de tester, ou accepter explicitement que
le test sera annulé au prochain cycle de reconciliation (par défaut toutes les
3 minutes).
