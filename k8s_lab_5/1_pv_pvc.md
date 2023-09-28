# PV, PVC

## PersistentVolume

Créer un PersistentVolume avec les caractéristiques suivantes :
* Nom : pv1
* Mode d'accès :ReadWriteOnce
* Capacité : 10Gi
* Type Local Storage dont le répertoire de persistance des données  est `/opt`
* Faite en sorte que ce dernier soit créé par affinité uniquement sur les Worker nodes `k8s-worker-1` et `k8s-worker-2` 
  
Remarquez les idfférents status des PV

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: 
  local:
    path: /opt
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-worker-1
          - k8s-worker-2
```

```bash
kubectl apply -f pv1.yml
persistentvolume/pv1 created

kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                STORAGECLASS   REASON   AGE
els-pv-volume   10Gi       RWO            Retain           Released    default/elasticsearch-data-quickstart-es-default-0                           27d
pv1             10Gi       RWO            Delete           Available                                                                                6s
```

</details>

## PersistentVolumeClaim

Créer un PersistentVolumeClaim qui consomera les le PV précédement créé avec les caractéristiques suivantes :
* Nom : pvc1
* Mode d'accès :ReadWriteOnce
* Capacité réclamé : 3Gi

Remarquez les idfférents status des PV et PVC

<details><summary>Correction</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

```bash
kubectl apply -f pvc1.yml
persistentvolumeclaim/pvc1 created

kubectl get pv,pvc
NAME                             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                                STORAGECLASS   REASON   AGE
persistentvolume/els-pv-volume   10Gi       RWO            Retain           Released   default/elasticsearch-data-quickstart-es-default-0                           27d
persistentvolume/pv1             10Gi       RWO            Delete           Bound      default/pvc1                                                                 117s

NAME                         STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pvc1   Bound    pv1      10Gi       RWO                           7s
```

```bash
kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                                STORAGECLASS   REASON   AGE
els-pv-volume   10Gi       RWO            Retain           Released   default/elasticsearch-data-quickstart-es-default-0                           27d
pv1             10Gi       RWO            Delete           Bound      default/pvc1                                                                 2m35s

kubectl get pvc
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc1   Bound    pv1      10Gi       RWO                           47s
```

</details>

## Création de Pod

Créer un Pod Nginx qui va reclamer de la volume à l'aide du PVC précédement créé
* Mettez dans le répertoire /opt des Worker node un fichier html
* Vérifier que lorsque le Pod est lancé la page web affichée est bien le fichier html mis dans /opt

<details><summary>Correction</summary>

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pvlab
spec:
  containers:
    - name: pvlab-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: new-storage
  volumes:
    - name: new-storage
      persistentVolumeClaim:
        claimName: pvc1
```

```bash
kubectl apply -f pvlab-pod.ymal
pod/pvlab created

kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
frontend-86d67d9884-5f6zr   1/1     Running   0          117m
frontend-86d67d9884-dftl2   1/1     Running   0          117m
pvlab                       1/1     Running   0          45s
robots-68548b4489-7hrrn     1/1     Running   0          52m
robots-68548b4489-tqvjw     1/1     Running   0          52m
```

PVC passe de Status Pending à Bound 

```bash
kubectl get pods,sc,pv,pvc 
```

</details>

Accédez à l'intérieur du pod et créez ou modifiez un fichier contenu dans le répertoire `/usr/share/nginx/html`. Remarquez que le fichier est bien présent dans /opt du Host après la suppression du Pod

<details><summary>Correction</summary>

```bash
kubectl exec -it pvlab -- sh
# touch /usr/share/nginx/html/samplefile 

# sortez du pod
$ ls -l /opt/
```

```bash
kubectl exec -it pvlab -- sh
# ls /usr/share/nginx/html
VBoxGuestAdditions-6.1.34  cni	containerd
# touch /usr/share/nginx/html/file-from-pod-pvlab
```

sur le node on voit bien que le ficher `file-from-pod-pvlab` aparait

```bash
vagrant@k8s-worker-2:~$ ls /opt/
VBoxGuestAdditions-6.1.34  cni  containerd  file-from-pod-pvlab
```

</details>