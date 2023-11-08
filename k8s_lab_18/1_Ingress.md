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

</details>