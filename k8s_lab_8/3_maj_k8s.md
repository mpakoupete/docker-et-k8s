# Mise à jour d'un cluster K8s

La Mise à jour d'un cluster K8s ([doc](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)) doit se faire dans l'ordre suivant :

* Nodes du Controle plane
* Worker nodes

Cette mise à jour consiste à mettre à niveau tous les composants qui composent le cluster et les outils d'administrations: 

* Kubeadm
* Kubelet
* Kubectl

## Mise à jour des Master nodes d'abords

### Mise à jour de Kubeadm

Kubeadm est notre outil d'aministration pour boostrapper un cluster kubernetes. De ce fait il sera la premier composant à être upgrader:

vérifiez la version actuelle

```bash
kubeadm version -o json
```

Quelles sont les nouvelles versions disponibles (https://kubernetes.io/releases/) ? Nous allons faire une MàJ à la version **1.28.0** sachant que la dernière est la 1.28.1

Nous allons donc installer la version correspondante de kubeadm.

Vous pouvez lancer la commande suivante kubeadm pour obtenir des suggestions de mise à niveau.

```bash
sudo kubeadm upgrade plan
```

```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.28.0-00 && \
sudo apt-mark hold kubeadm
```

Appliquez la mise à jour à l'aide de la commande suivante.

```bash
sudo kubeadm upgrade apply v1.28.0
```

Consultez [cette section](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#how-it-works) pour comprendre ce que cette command d'upgrade implique.

### Mise à jour de Kubelet et Kubectl

Mettre à jour le kubelet et le kubectl et redémarrer le service kubelet

```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet='1.28.0-00' kubectl='1.28.0-00' && \
apt-mark hold kubelet kubectl
Restart the kubelet:

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## Mise à jour des Worker nodes

On procède un peu différemment que précédemment mais l'idée reste le même:

### Mise à jour de Kubeadm

vérifiez la version actuelle

```bash
kubeadm version -o json
```

Nous allons également installer la version correspondante de kubeadm.

```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm=1.28.0-00 && \
sudo apt-mark hold kubeadm
```

Appliquez la mise à jour à l'aide de la commande suivante. (différente de la précédente)

```bash
sudo kubeadm upgrade node
```

Consultez [cette section](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#how-it-works) pour comprendre ce que cette command d'upgrade du node implique.

### Mettre le node en maintenance

Etant donnée que la MàJ nécessite le redemarrage du service Kubelet, pour une bonne gestion, il faudra evincer les pods et marquer le node non schedulable.

```bash
kubectl drain node01 --ignore-daemonsets
```

### Mise à jour de Kubelet et Kubectl

Mettre à jour le kubelet et le kubectl et redémarrer le service kubelet

```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet='1.28.0-00' kubectl='1.28.0-00' && \
apt-mark hold kubelet kubectl
Restart the kubelet:

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Sortir le node de la maintenance

Pour marquer le node schedulable à nouveau :

```bash
kubectl uncordon node01
```

Répéter le scénario sur les autres worker nodes