# Microservices

Nous nous mettons dans le contexte où nous avons développé une application `example-voting-app`
Pour rappel, [example-voting-app](https://github.com/dockersamples/example-voting-app) est une application maintenu par la communauté Docker qui est très souvent utilisée pour faire des démo de microservices.
Il est concu en microservices avec l'architecture suivante :

![architecture](https://user-images.githubusercontent.com/59444079/202094328-30a5cb07-f33b-40bf-828a-7dc5de405bb7.png)

La stack est composée de 5 services:
* un service front `vote` développé en Python : Les utilisateurs peuvent y accéder pour voter en cliquant
* un service front `result` développé en Node.js : Les utilisateurs peuvent y accéder pour voir les résultats du vote
* un service backend `worker` qui traites les requètes et les persistent dans une base de donnée PostgreSQL
* un service de Base de données `db` PostgreSQL qui persiste la donnée
* un service de cache `redis` utilisé par le microservice `voting-app`

L'objectif du Lab sera de Dockeriser cette stack et la déployer sur un cluster Kubernetes. Les différentes notions qui seront abordées:
* ConfigMap
* Secrets
* PV, PVC, SC
* Ingress
* GiOps avec ArgoCD

## Partie 1 : Mettre en place des autres microservices

Créez un manifest de déploiement pour chacun des microservices
`vote`, `result`, `worker`, `redis`, `db` et exposez ces déployment à l'aide des services qui porteront le même nom.

Voir le répertoire `solution`

<details><summary>Correction</summary>

Accéder au répertoire `solution`

```bash
kubectl apply -f .
```

</details>

## Partie 3 : Ingress

Installer l'ingress controller `nginx` (https://kubernetes.github.io/ingress-nginx/deploy/) et configurer les règle d'Ingress pour que les redirections suivantes soient faites :

* www.k8s-lab.local/vote      ---> service `vote`
* www.k8s-lab.local/result    ---> service `result`
* par défaut www.k8s-lab.local      ---> service `vote`

<details><summary>Correction</summary>

La première chose sera d'abord d'installer un Ingress-controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

Créer l'ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: voting-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: www.k8s-lab.local
      http:
        paths:
          - path: /vote
            pathType: Prefix
            backend:
              service:
                name: vote
                port:
                  number: 5000
          - path: /result
            pathType: Prefix
            backend:
              service:
                name: result
                port:
                  number: 5001
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vote
                port:
                  number: 5000
```

<details>

## Partie 4 : GitOps

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

Patcher le service si vous être sur le Cloud

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