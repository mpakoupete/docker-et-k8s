# Services

## Accès au déploiement frontend - ClusterIP

Dans le Lab consacré au déployement, nous avions déployé Frontend. Comment allez-vous accéder à ce déploiement ?

<details><summary>Correction</summary>

Un déploiement dans Kubernetes n'a du sens que pour le Control plane. Derrière le déploiement en réalité c'est des Pods/Conteneurs qui seront tangibles aux quels les utilisateurs devront accéder. Donc de ce fait pour accéder au déploiement créé, il suffira en fait d'accéder aux Pods/conteneurs individuels créés, et donc par leurs IPs. avec l'option `-o wide` ou le Describe vous pouvez obtenur leurs IPs.

Bien évidemment, c'est fastidieux car il faut manuellement les chercher.
Pour rendre facile l'accès de ces 3 Pods/conteneurs créés nous aurons besoin d'un object Service qui va configurer une sorte de loadbalancer entre les 3 pods.

Désormais donc, au lieu de chercher les iP des 3 conteneurs, il nous suffira d'accéder au `endpoint du service` et le tour est joué.

</details>

Créer un service du nom de `frontend-http` de type ClusterIP qui expose le Déploiement `frontend` au port 80

<details><summary>Correction</summary>

```bash
kubectl expose deployment frontend --port=80 --target-port=80 --name=frontend-http
```

</details>


Listez les services

<details><summary>Correction</summary>

```bash
kubectl get service
```

</details>

Tester l'accès à votre déploiement avec un Pod que vous aller nommer `client` dont l'image est `curlimages/curl`

<details><summary>Correction</summary>

```bash
kubectl run client --rm  -it --image curlimages/curl -- sh
$ curl http://frontend-http
```

</details>

Kubectl permet de faire une [sorte de tunneling/forwarding des requêtes](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) pour accéder à un endpoint interne au cluster (Ex: IP de Pod, Service de type ClusterIP...)
Ex :

```bash
kubectl port-forward service/frontend-http 8503:80

Sur le navigateur de votre Client faite un `http://localhost:8503`
```

Comment expliquez vous le mécanisme de connexion entre le Pod `client` et les Pods du deployment `frontend` ? En d'autres termes comment expliquez vous la résolution du nom `frontend-http` ?

<details><summary>Correction</summary>

```bash
vagrant@k8s-master-1:~$ kubectl get all -n kube-system
NAME                                           READY   STATUS    RESTARTS       AGE
pod/calico-kube-controllers-566654d67d-lrgpn   1/1     Running   0              20h
pod/calico-node-gk6qg                          1/1     Running   5 (20h ago)    40d
pod/calico-node-ljmmc                          1/1     Running   6 (20h ago)    40d
pod/calico-node-qg7h7                          1/1     Running   5 (20h ago)    40d
pod/coredns-565d847f94-6lfhq                   1/1     Running   6 (20h ago)    40d
pod/coredns-565d847f94-bnqlv                   1/1     Running   6 (20h ago)    40d
pod/etcd-k8s-master-1                          1/1     Running   11 (20h ago)   40d
pod/kube-apiserver-k8s-master-1                1/1     Running   17 (20h ago)   40d
pod/kube-controller-manager-k8s-master-1       1/1     Running   2 (20h ago)    28d
pod/kube-proxy-g9s6l                           1/1     Running   5 (20h ago)    40d
pod/kube-proxy-m29bk                           1/1     Running   5 (20h ago)    40d
pod/kube-proxy-z7kn6                           1/1     Running   6 (20h ago)    40d
pod/kube-scheduler-k8s-master-1                1/1     Running   0              177m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   40d

NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/calico-node   3         3         3       3            3           kubernetes.io/os=linux   40d
daemonset.apps/kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   40d

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calico-kube-controllers   1/1     1            1           40d
deployment.apps/coredns                   2/2     2            2           40d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/calico-kube-controllers-566654d67d   1         1         1       40d
replicaset.apps/coredns-565d847f94                   2         2         2       40d
```

Dans notre environnement Kubernetes, il y'a un deploiment du nom de coredns qui implemente un DNS dans le Kubernetes. 
Quand une ressources Service est créé, une IP virtuelle est créé lui correspondant et l'entré DNS correspondante est ajouté dans le DNS CoreDNS. 
Kube-proxy & Kubelet se chargent dattribuer à chaque Pod créés le fichier `/ets/resolv.conf` qui indique le service DNS à contacter pour la résolution.
Kube-proxy ajoutes des règles de routage sur chaque neoud pour rediriger le traffic en direction de ce VirtualIP vers le IP des Pod correspondant au service. C'est de cette manière quà partir de n'importe quel Pod on peut accéder à un service.

```bash
vagrant@k8s-master-1:~$ kubectl run client --rm  -it --image curlimages/curl -- sh
If you don't see a command prompt, try pressing enter.
/ $ cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
/ $
```

Vous remarquerez que nous avons maintenant juste besoin du nom du service qui joue d'office le role de nom DNS de notre déploiement.

</details>

## Accès au déploiement frontend - NodePort

Créer un manifest YAML du service du nom de `web-service` de Type NodePort qui exposera le déploiement frontend sur les port 31008 des noeuds

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: frontend
  type: NodePort
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 31008

```

```bash
kubectl create -f web-service.yaml 
```

</details>

Visualisez les details de ce service

<details><summary>Correction</summary>

```bash
kubectl get services
kubectl describe service web-service
```
On note bien la distinction : TargetPort, Port, NodePort

</details>

## Accès au déploiement frontend - Loadbalancer

Créer un manifest YAML du service du nom de `lb-service` de Type LoabBalancer qui exposera le déploiement frontend sur les port 31008 des noeuds

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lb-service
spec:
  selector:
    tier: frontend
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80

```

```bash
kubectl create -f lb-service.yaml 
```

</details>

Visualisez les details de ce service

<details><summary>Correction</summary>

```bash
kubectl get services
kubectl describe service lb-service
```
On note bien la distinction : TargetPort, Port, NodePort

</details>

Quel est le status de ce service ? Pourquoi

<details><summary>Correction</summary>

Nous n'avons pas de controlleur capable de créer un Loadbalancer externe. Il est préférable d'avoir une infrastructure approprié pour ce type de service ou cela est réalisable dans un Cluster K8s d'un Clou Provider : GKE, AKS, EKS,...

</details>