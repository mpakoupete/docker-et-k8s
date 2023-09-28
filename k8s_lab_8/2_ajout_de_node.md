# Ajout de nodes au Cluster

Il se trouve que vous avez besoin d'ajouter plus de ressources de calcul à votre cluster et pour cela vous souhaiter ajouter un nouveau noeud pour avoir 3 worker nodes.
De manière basique pour l'ajout d'un noeud à un cluster, on a besoin de :

* Installer le Conteneur Runtime : **CRI-O**
* Installer **Kubelet** l'agent kubernetes à la même version que Kubernetes existant
* Installer **Kubeadm** l'outil pour boostrapper un node (master ou worker) à la même version que Kubernetes existant

### Préparation du node 3

Il nous faut lancer un noeud vierge.
Accéder au répertoire `new_node` et lancer `vagrant up` ensuite se connecter à la machine `vagrant ssh <nom_vm>`

### Confirgration de base commune à tous les hôtes


Configurer le DNS : 

```bash
sudo mkdir /etc/systemd/resolved.conf.d/
cat << _EOF | sudo tee /etc/systemd/resolved.conf.d/dns_servers.conf
[Resolve]
DNS=8.8.8.8
_EOF
```

Redémarrage du service de dns.

```bash
sudo systemctl restart systemd-resolved
```

Désactivation du swap

```bash
sudo swapoff -a
# commenter les ligne swap dans le fichier fstab
```

### Installation du Container runtime CRI-O

```bash
sudo apt-get update -y
```

```bash
OS=xUbuntu_22.04
KUBERNETES_VERSION=1.27.1-00
VERSION="$(echo ${KUBERNETES_VERSION} | grep -oE '[0-9]+\.[0-9]+')"
```

Créer le fichier .conf pour charger les modules au démarrage

```bash
cat << _EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
_EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure les paramètres sysctl requis, ceux-ci persistent à travers les redémarrages.

```bash
cat << _EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
_EOF

sudo sysctl --system
```

Configuration des dépôts de paquets Debian

```bash
cat << _EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
_EOF
```

Configuration des dépôts de paquets Debian pour cri-o

```bash
cat << _EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
_EOF
```

Configuration des clé GPG pour la vérification et l'authentification des paquets

```bash
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
```

Installation de CRI-O

```bash
sudo apt-get update
sudo apt-get install cri-o cri-o-runc -y
```

Reload et redemarrage du service

```bash
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

### Installation de Kubelet & Kubeadm

Installation des paquets utils pour l'installation de Kubelet & Kubeadm

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
```

Installation de kubelet & kubeadm

```bash
sudo apt-get install -y kubelet="$KUBERNETES_VERSION" kubectl="$KUBERNETES_VERSION" kubeadm="$KUBERNETES_VERSION"

sudo apt-get update -y
sudo apt-get install -y jq
```

## joindre le node 3 au cluster

Pour joindre le node 3 au cluster il faut se rendre sur le master node pour générer la commande de jonction.

sur le master node :

```bash
sudo kubeadm token create --print-join-command
```

Le résultat généré est la commande que vous devez exécuter sur le node3.

Une fois la commande exécutée, retournez sur le master node pour constater l'ajout.

```bash
kubectl get nodes
```
