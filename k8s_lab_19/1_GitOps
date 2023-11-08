## GitOps

L'objectif de cette partie du lab est d'automatiser le Déploiement de la stack de microservice suivant le shéma workflow suivant :
La mise à jour des fichiers de déploiement du microservice dans le dépôt GitHub ==> déclenche le déploiement de cette nouvelle mise à jour dans le cluster Kubernetes GKE.
Pour ce faire, nous utiliserons ArgoCD comme outil de Continuous Delivery (CD)

### Installer ArgoCD

De manière natif K8s de permet pas de faire du CD ==> Solution, nous allons étendre son API avec l'installation de ArgoCD pour ajouter cette fonctionalité

```bash
kubectl api-resources | grep -i argo
```

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Patcher le service si vous être sur un Clouster K8s Cloud

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Récupérer le mot de pass Admin
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Via l'IP du loadbalancer ou NodePort, vérifiez que vous avez bien accès la console d'administration et de gestion de ArgoCD

On utilisera `ArgoCD CLI` pour certaines commandes d'administration; pour ce faire vous devez l'installer sur votre machine d'administration.

Exemple d'installation sur MacOS
```bash
brew install argocd
```

Se connecter à ArgoCD

```bash
kubectl get service argocd-server -n argocd --output=jsonpath='{.status.loadBalancer.ingress[0].ip}'
# argocd login <IP LoadBalancer> --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo) --insecure

argocd login $(kubectl get service argocd-server -n argocd --output=jsonpath='{.status.loadBalancer.ingress[0].ip}') --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo) --insecure

```

Ajout du repos Git

Créer un Repo Github et ajoutez le dans ArgoCD comme Repo à partir duquel ils déploiera l'application `voting-app`

```bash
argocd repo add git@github.com:mpakoupete/gitops-test-argocd.git --ssh-private-key-path ~/.ssh/id_rsa
```

Aller dans l'application (interface Web de ArgoCD) poour les config
- Application

Configurer un déploiement continu de l'application dans le namespace dev-voting-app

```bash
kubectl create ns dev-voting-app
```