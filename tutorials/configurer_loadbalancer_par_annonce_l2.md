# Configurer un Service LoadBalancer avec Cilium (sans utiliser MetalLB)

- [Configurer un Service LoadBalancer avec Cilium (sans utiliser MetalLB)](#configurer-un-service-loadbalancer-avec-cilium-sans-utiliser-metallb)
  - [Comment fonctionne l'annonce L2](#comment-fonctionne-lannonce-l2)
  - [Scénario](#scénario)
  - [Configurer les noeuds elligibles pour les réponses ARP](#configurer-les-noeuds-elligibles-pour-les-réponses-arp)
  - [Définir la plage d'IP utilisable par les services LoadBalancer](#définir-la-plage-dip-utilisable-par-les-services-loadbalancer)
  - [Vérifier le fonctionnement](#vérifier-le-fonctionnement)

## Comment fonctionne l'annonce L2

L' **Annonce L2** est une fonctionnalité qui rend les services visibles et accessibles sur le **réseau local**. Cette fonctionnalité est principalement destinée aux déploiements *on-premise* au sein de réseaux sans routage BGP.

Lorsqu'elle est utilisée, cette fonctionnalité répondra aux requêtes ARP à destination des adresses de type *LoadBalancer*. Ces IP sont des IP virtuelles (non déployées sur les interfaces réseaux) sur plusieurs nœuds. Pour chaque service, un nœud sera élu pour répondra aux requêtes ARP et renverra son adresse MAC. Ce nœud effectuera l'équilibrage de charge basé sur la configuration du service, agissant ainsi comme un *load balancer north/south*.

Cette méthode permet de dédier une adresse IP propre pour un service donné et de ne pas être limité sur la plage de ports utilisables (contrairement au service *NodePort*). Cette caractéristique est particulièrement pertinente dans le cadre de *reverse proxy* qui doivent écouter sur les ports tradtionnels (80/443 par exemple).

## Scénario

Supposons un service ingress basé sur `nginx` basé dans le namespace `ingress-nginx`. Il n'y a que un service `LoadBalancer` dans ce namespace. Nous souhaitons associer l'IP `192.168.122.10` au service `LoadBalancer` et reserver l'annonce L2 au noeud `node1`.

## Configurer les noeuds elligibles pour les réponses ARP

Tout d'abord, il est nécessaire de configurer le/les noeud(s) elligible(s) pour réaliser les réponses ARP.

```bash
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: cilium-lb-ingress
  namespace: kube-system
spec:
  # On sélectionne le/les noeud(s) elligible(s)
  # Dans notre cas d'usage, il s'agit de `node1`
  nodeSelector:
    matchLabels:
      kubernetes.io/hostname: "node1"
  # On sélectionne le/les services à annoncer (sous réserve qu'il y ait un service LoadBalancer)
  # Dans notre cas d'usage, on va utiliser le namespace pour discriminer le service nginx
  # Dans le cas où il y a plusieurs services LoadBalancer dans ce namespace, l'ensemble des services seront annoncés
  serviceSelector:
    matchLabels:
       "io.kubernetes.service.namespace": "ingress-nginx"
  loadBalancerIPs: true
```

```bash
kubectl apply -f <mon_script>
```

## Définir la plage d'IP utilisable par les services LoadBalancer

Après avoir configurer les noeuds annonceurs, il faut définir la plage d'IPs utilisable pour associer une IP au service `LoadBalancer`.

```bash
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: ingress-pool
  namespace: kube-system
spec:
  # Nous voulons utiliser l'IP 192.168.122.10 pour le service
  # Nous définissons une plage composée que d'une IP car il n'y a que un service à couvrir
  blocks:
  - start: "192.168.122.10"
    stop: "192.168.122.10"
  # Nous associons la plage IP au service concerné
  # Dans notre cas d'usage, on va utiliser le namespace pour discriminer le service nginx
  # Soyez sûr d'avoir assez d'IPs pour couvrir l'ensemble des services LoadBalancer sélectionnés
  # Pour être sûr d'avoir une IP fixe associée à un service, configurer des fichiers <CiliumLoadBalancerIPPool> avec:
  # Une plage d'1 IP et une sélection qui isole un seul service.
  serviceSelector:
    matchLabels:
      "io.kubernetes.service.namespace": "ingress-apisix"
```

```bash
kubectl apply -f <mon_script>
```

## Vérifier le fonctionnement

Nous constatons que le service `ingress-nginx-controller` est associé au `node1`.

```bash
kubectl -n kube-system get lease

NAME                                                       HOLDER   AGE
[...]
cilium-l2announce-ingress-nginx-ingress-nginx-controller   node1    25d
```

Nous constatons que notre plage d'IP est active, sans conflit avec d'autres plages et qu'aucune IP n'est disponible. Cette valeur est normale car la seule IP disponible a été allouée au service `nginx`.

```bash
kubectl get CiliumLoadBalancerIPPool -A
NAME                 DISABLED   CONFLICTING   IPS AVAILABLE   AGE
pool-nginx-ingress   false      False         0               25d
```

Nous constatons que l'IP `192.168.122.10` a bien été associée au service `nginx`.

```bash
kubectl get service -n ingress-nginx

kubectl get service -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.110.64.199    192.168.122.10   80:31275/TCP,443:30377/TCP   25d
ingress-nginx-controller-admission   ClusterIP      10.106.244.238   <none>           443/TCP                      25d
```
