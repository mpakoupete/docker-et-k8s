# initContainer et gestion des ressources CPU RAM

## Init Container

[Conteneurs init](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) : _"conteneurs spécialisés qui s'exécutent avant les conteneurs d'applications dans un Pod. Les conteneurs d'initialisation peuvent contenir des utilitaires ou des scripts d'installation qui ne sont pas présents dans une image d'application."_

Créer un Pod du nom de `init-pod` qui contient un initContainer dont le but est de télécharger la page web suivante : https://example.com/ et la sauvegarde `index.html` et qui sera servi par le container principal `nginx`.


<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx 
    ports:
    - containerPort: 80
    volumeMounts:
    - name: sharedir
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/share-dir/index.html"
    - https://example.com/
    volumeMounts:
    - name: sharedir
      mountPath: "/share-dir"
  volumes:
  - name: sharedir
    emptyDir: {}
```

</details>

Exposez le Pod via un service de type NodePort et testez l'accès.

## Requests & Limits

Dans le soucis de mieux gérer les resources du cluster vous souhaitez appliquer ces mesures au conteneur précédemment créé :
* Resource réservé de 10% d'un CPU - limité à 25% d'un CPU
* Resource réservé de 128Mi de RAM - limité à 256Mi de RAM
Apportez les modifications nécessaires

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi  
    ports:
    - containerPort: 80
    volumeMounts:
    - name: sharedir
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: install
    image: busybox
    command:
    - wget
    - "-O"
    - "/share-dir/index.html"
    - https://example.com/
    volumeMounts:
    - name: sharedir
      mountPath: "/share-dir"
  volumes:
  - name: sharedir
    emptyDir: {}
```

</details>