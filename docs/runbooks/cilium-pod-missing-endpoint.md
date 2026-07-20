# Pod applicatif : DNS/réseau cassés en permanence malgré un DaemonSet Cilium sain

## Symptôme

Un pod précis (souvent un composant du chart Cilium lui-même, ex: `hubble-relay`,
`hubble-ui`) reste bloqué en `CrashLoopBackOff`/`0/1 Ready` avec des erreurs de
résolution DNS ou de connexion réseau vers des Services internes, alors que :

- le `DaemonSet cilium` est `N/N Ready` sur tous les nodes,
- CoreDNS répond correctement (`nslookup` depuis un pod fraîchement créé sur le
  même node fonctionne),
- aucune `NetworkPolicy`/`CiliumNetworkPolicy` ne bloque quoi que ce soit.

```bash
kubectl -n kube-system logs -l k8s-app=hubble-relay --tail=20
# "dns: A record lookup error: ... i/o timeout" en boucle, alors que le service
# et ses endpoints existent et résolvent normalement pour tout le reste du cluster
```

## Diagnostic

Comparer l'IP du pod cassé avec la liste des endpoints connus de l'agent
Cilium tournant sur le même node :

```bash
kubectl -n kube-system get pod -l k8s-app=hubble-relay -o wide
# IP: 10.244.11.2, NODE: k8s-poc-worker07

CILIUM_POD=$(kubectl -n kube-system get pods -l k8s-app=cilium \
  --field-selector spec.nodeName=k8s-poc-worker07 -o jsonpath='{.items[0].metadata.name}')

kubectl -n kube-system exec $CILIUM_POD -c cilium-agent -- cilium-dbg endpoint list \
  | grep -E "10.244.11.2|IDENTITY"
```

Si l'IP du pod cassé n'apparaît **nulle part** dans `cilium-dbg endpoint list`,
c'est confirmé : le pod n'a jamais reçu d'endpoint Cilium.

## Cause

Race condition au bootstrap (ou à un redémarrage de l'agent Cilium sur ce
node) : le pod a été créé par kubelet, qui a invoqué le plugin CNI Cilium au
même moment où l'agent local n'était pas encore prêt à traiter la requête (ou
l'appel a échoué/timeout). Kubelet ne réessaie **pas** l'appel CNI pour un pod
déjà créé — le pod garde un network namespace vide de tout programme BPF pour
toujours, jusqu'à ce qu'il soit recréé.

Rencontré spécifiquement sur `hubble-relay` et `hubble-ui` lors du premier
bootstrap de Cilium sur `k8s-poc` (bootstrap manuel via `bootstrap.sh`, avant
adoption complète par ArgoCD) — ces pods (Deployments avec 1 seule replica,
sans anti-affinity ni rolling update en cours) sont particulièrement exposés
car rien ne force leur recréation naturellement après le bootstrap initial.

## Fix

Supprimer le(s) pod(s) concerné(s) — le Deployment/DaemonSet les recrée, et
cette fois l'appel CNI aboutit puisque l'agent Cilium du node est stable
depuis longtemps :

```bash
kubectl -n kube-system delete pod -l k8s-app=hubble-relay
kubectl -n kube-system delete pod -l k8s-app=hubble-ui

# vérifier l'endpoint après recréation
kubectl -n kube-system get pod -l k8s-app=hubble-relay -o wide
kubectl -n kube-system exec $CILIUM_POD -c cilium-agent -- cilium-dbg endpoint list | grep hubble-relay
```

## Prévention

Après tout bootstrap manuel de Cilium sur un nouveau cluster (`bootstrap.sh`),
avant de déclarer la stack saine, vérifier que **chaque pod du namespace
`kube-system`** (pas seulement le DaemonSet `cilium` lui-même) a un endpoint
Cilium correspondant :

```bash
kubectl -n kube-system get pods -o wide | awk 'NR>1{print $6}' | sort -u > /tmp/pod-ips.txt
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  CILIUM_POD=$(kubectl -n kube-system get pods -l k8s-app=cilium --field-selector spec.nodeName=$node -o jsonpath='{.items[0].metadata.name}')
  kubectl -n kube-system exec $CILIUM_POD -c cilium-agent -- cilium-dbg endpoint list
done > /tmp/cilium-endpoints.txt
# toute IP de /tmp/pod-ips.txt absente de /tmp/cilium-endpoints.txt = pod à recréer
```

Un `ArgoCD Application` de statut `Degraded` sur `cilium` malgré un
DaemonSet visiblement sain doit systématiquement faire suspecter ce cas avant
toute autre hypothèse.
