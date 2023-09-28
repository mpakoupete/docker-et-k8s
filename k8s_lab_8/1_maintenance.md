# Opération maintenance d'un noeud du cluster

Afficher les nodes de votre cluster

```bash
kubectl get nodes
```

Il se trouve que vous souhaiter mettre à jour le node `k8s-worker-1` qui necessitera une indisponibilité à cause du redémarrage, pour cela certaines précautions peuvent être mise en place comme : 

- évencer les Pods de ce Noeud
- le rendre non schedulable afin que les nouveaux pods ne soient pas schedulé sur le node en question.

Comment le feriez vous ?


<details><summary>Correction</summary>

_"Vous pouvez utiliser kubectl drain pour expulser en toute sécurité tous vos pods d'un nœud avant d'effectuer une maintenance sur le nœud (par exemple, une mise à niveau du noyau, une maintenance matérielle, etc.) Les expulsions sécurisées permettent aux conteneurs du pod de se terminer de manière élégante. "_ [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#use-kubectl-drain-to-remove-a-node-from-service)

```bash
kubectl drain k8s-worker-1
```

Si des daemonsets sont en cours d'execution il vous sortira une erreur

```bash
$ kubectl drain k8s-worker-1 --ignore-daemonsets
node/k8s-worker-1 already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-xzp6z, kube-system/kube-proxy-k5kjl
evicting pod test/frontend-7866b45dcd-kwjbv
evicting pod kube-system/calico-kube-controllers-566654d67d-zphqm
evicting pod default/frontend-5bbbfcf84f-vxc95
pod/calico-kube-controllers-566654d67d-zphqm evicted
pod/frontend-7866b45dcd-kwjbv evicted
pod/frontend-5bbbfcf84f-vxc95 evicted
node/k8s-worker-1 evicted
```

Vous pouvez afficher les événements du cluster

```bash
$ kubectl get events
LAST SEEN   TYPE     REASON               OBJECT                           MESSAGE
3m9s        Normal   Scheduled            pod/frontend-5bbbfcf84f-qd9zn    Successfully assigned default/frontend-5bbbfcf84f-qd9zn to k8s-worker-3
3m9s        Normal   Pulled               pod/frontend-5bbbfcf84f-qd9zn    Container image "nginx:1.14.2" already present on machine
3m9s        Normal   Created              pod/frontend-5bbbfcf84f-qd9zn    Created container nginx
3m9s        Normal   Started              pod/frontend-5bbbfcf84f-qd9zn    Started container nginx
3m9s        Normal   Killing              pod/frontend-5bbbfcf84f-vxc95    Stopping container nginx
3m9s        Normal   SuccessfulCreate     replicaset/frontend-5bbbfcf84f   Created pod: frontend-5bbbfcf84f-qd9zn
3m31s       Normal   NodeNotSchedulable   node/k8s-worker-1                Node k8s-worker-1 status is now: NodeNotSchedulable
```

</details>

La conséquence de cette commande précédante c'est que les Pods qui étaient en cours d'execution sont évincés du node et reschedulés ailleurs et ensuite le Node est marqué avec un Taint : `Noschedule` pour empêcher tout nouveau Pod d'y être placé par le `Kube-Scheduler`

```bash
$ kubectl describe nodes k8s-worker-1

Name:               k8s-worker-1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    (...)
Taints:             node.kubernetes.io/unschedulable:NoSchedule
Unschedulable:      true
(...)
```

Une fois que la maintenance est terminée on marque maintenant le Node comme schedulable afin qu'il commence à recevoir les Pods

```bash
kubectl uncordon k8s-worker-1
```

kubectl get events
LAST SEEN   TYPE     REASON               OBJECT                           MESSAGE
(...)
8s          Normal   NodeSchedulable      node/k8s-worker-1                Node k8s-worker-1 status is now: NodeSchedulable