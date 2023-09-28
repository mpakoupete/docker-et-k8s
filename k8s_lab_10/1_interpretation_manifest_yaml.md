# Comprendre un Manifest YAML

Dans cette partie il sera question de voir ensemble comment lire un fichier YAML.
Il arrive très souvent de recevoir des fichier YAML provenant de fournisseurs ou partenaires que nous devont déployer dans notre Cluster. Il est toujours interessant de savoir ce que ce fichier apportera comme modification dans notre environnement.

Nous prendrons comme exemple Le lancement de Filebeat dans un Cluster Kubernetes

Lien du Manifest : https://raw.githubusercontent.com/elastic/beats/8.4/deploy/kubernetes/filebeat-kubernetes.yaml

Télécharger ce fichier YAML et expliquer le, section par section

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

    # To enable hints based autodiscover, remove `filebeat.inputs` configuration and uncomment this:
    #filebeat.autodiscover:
    #  providers:
    #    - type: kubernetes
    #      node: ${NODE_NAME}
    #      hints.enabled: true
    #      hints.default_config:
    #        type: container
    #        paths:
    #          - /var/log/containers/*${data.kubernetes.container.id}.log

    processors:
      - add_cloud_metadata:
      - add_host_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.4.3
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: changeme
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          # If using Red Hat OpenShift uncomment this:
          #privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0640
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          # When filebeat runs as non-root user, this directory needs to be writable by group (g+w).
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: filebeat
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: kube-system
roleRef:
  kind: Role
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: filebeat-kubeadm-config
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: kube-system
roleRef:
  kind: Role
  name: filebeat-kubeadm-config
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups: ["apps"]
  resources:
    - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources:
    - jobs
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: filebeat
  # should be the namespace where filebeat is running
  namespace: kube-system
  labels:
    k8s-app: filebeat
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs: ["get", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: filebeat-kubeadm-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - kubeadm-config
    verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
---
```

<details><summary>Correction</summary>

```yaml
# les "---" représentent à chaque fois le début d'un Fichier YAML. 
# Nous comprenons qu'il y'a 9 resources Kubernetes à déployer ou encore 9 "fichier" YAML contenus dans ce seul fichier.
# Une astuce pour bien lire ce genre de fichier est de chercher à voir quel type de Workload resource on veux déployer ? Est-ce : une Pod? Un Déploiement ? un StatefulSet ? un DaemonSet ?
# une fois que vous avez répéré de quoi il s'agit ma recommandation est de commencer par ce manifest. Dans ce cas précis, il s'agit d'un DaemonSet. Donc commençons par là. Pourquoi ?
# Parce que Kubernetes est un orchestrateur de conteneurs et l'objectif d'utiliser Kubernetes ce n'est pas de jouer avec tout type de resources sans aucun but derrière. Mais plutôt de faire en sorte que chaque ressource qui sera créer réponde à un But: celui de déployer des Pods pour assurer le service voulu dans un environnement sécurisé désiré.
# Donc commencer par le DaemonSet nous donne déjà une idée de ce que ce manifeste fera: Assurer la mise en place d'un Pod qui tournera sur chaque noeud (Revoir ce que c'est qu'un [DaemonSet ici](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/))

# Liser les commentaires dans les manifestes dans l'ordre suivant :
#      1. manifest de DaemonSet ==> Allons y 
#      2. Manifest de ServiceAccount ==> Allons y 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-          # Lorsqu'un configMap débute de la sorte ==> alors c'est le contenu du fichier qui est mis. et ce fichier peut être monté dans un Pod
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

    # To enable hints based autodiscover, remove `filebeat.inputs` configuration and uncomment this:
    #filebeat.autodiscover:
    #  providers:
    #    - type: kubernetes
    #      node: ${NODE_NAME}
    #      hints.enabled: true
    #      hints.default_config:
    #        type: container
    #        paths:
    #          - /var/log/containers/*${data.kubernetes.container.id}.log

    processors:
      - add_cloud_metadata:
      - add_host_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
---
apiVersion: apps/v1
kind: DaemonSet # Il s'agit d'un DaemonSet
metadata:
  name: filebeat
  namespace: kube-system # le pods sera déployer dans le namespace kube-system
  labels:
    k8s-app: filebeat   # Ce DaemonSet portera ce label filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat # Le Selector du DaemonSet est les Template qui porteront ce label précis 
  template:         # Début du template
    metadata:
      labels:
        k8s-app: filebeat   # Le label auquel fait référence DaemonSet dans les Selector plus haut.
    spec:
      serviceAccountName: filebeat      # Le service account filebeat est le compte service avec le quel le pod va emettre ses requettes dans le Cluster. Quand on voit cette ligne, on devine du coup qu'ils doivent avoir inclu un manifest qui crée ce ServiceAccount filebeat. (c'est le Dernier Manifest. vous pous pouvez aller jeter un coup d'oeil et revenir)
      terminationGracePeriodSeconds: 30     # L'application à l'intérieur du Pod dispose de 30s de temps pour se terminer (terminationGracePeriodSeconds). Le minuteur démarre lorsque le hook de préarrêt est appelé ou lorsque le signal TERM est envoyé si aucun hook n'est défini. Si le processus est toujours en cours d'exécution après l'expiration de ce délai de grâce, il est arrêté de force par le signal KILL. Cela met fin au conteneur.
      hostNetwork: true     # Ce pod va s'exécuter dans le réseau hôte du node où il est déployé. La conséquence c'est que le pod pourra alors utiliser l'espace de noms du réseau et les ressources réseau du node, le pod peut accéder au "Lo", écouter les adresses et surveiller le trafic des autres pods sur le node.
      dnsPolicy: ClusterFirstWithHostNet # Ceci doit être explicitement spécifier quant l'option hostNetwork: true est utilisé. Ceci garantit le fait qu'il puisse résoudre les noms de services ou resources internes à Kubernetes. Par défaut, le fichier de configuration DNS vers lequel pointe l'indicateur --resolv-conf du kubelet est configuré pour les workload s'exécutant avec :hostNetwork. 
      containers:
      - name: filebeat  # le nom de notre conteneur
        image: docker.elastic.co/beats/filebeat:8.4.3   # l'image qui sera télécharger. avec la commande qui sera exécutée
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:                                      # On charge dans le Pod les variables d'environnements suivante. Piurcertaine variables la valeur est donnée immédiatement et pour NODE_NAME il est pris de l'environement système de Kubernetes.
        - name: ELASTICSEARCH_HOST
          value: elasticsearch
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: changeme
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        - name: NODE_NAME # pris de l'environement système de Kubernetes. Un pod peut accéder à certaires information fournissables par l'API Server
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:        # Le Pod s'exécutera avec les droit Root
          runAsUser: 0
          # If using Red Hat OpenShift uncomment this:
          #privileged: true
        resources:      # Le Pod s'exécutera avec une limitation de resources RAM et CPU mentionné. le Request c'est le minimum qui est exigé. la conséquene de ça est que si un node n'a pas suffisemnt de ressource minimale exigée le pod ne se lancera pas sur ce node
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:           # Des volumes sont déclarés
        - name: config          # Ce volume n'est pas un volume de donnée quelconque mais un fichier qui sera placé dans le Pod ==> Voir Spec.volumes 
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data            # Volume de données monté
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:                 # Les volumes montés sont déclarés ici.
      - name: config           # Ce volumes est particulier. En fait c'est un Config Map (le premier manifest) qui a déclare une variable en tant que fichier. Il sera monté avec les droits spécifiés
        configMap:
          defaultMode: 0640
          name: filebeat-config
      - name: varlibdockercontainers    # On monte un Directory du Node /var/lib/docker/containers dans le conteneur en suivant le même chemin (voir plus haut) => On imagine que le Node écrit constament dans données la dedans (CRE y met les logs des conteneurs par defaut)
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog                   # Pareil également On monte un Directory du Node /var/log dans le conteneur en suivant le même chemin (voir plus haut)
        hostPath:
          path: /var/log
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:                     # Le Pod persistera certaines données dans ce directory qui sera créé sur le node si ce dernier n'existe pas. (Option DirectoryOrCreate)
          # When filebeat runs as non-root user, this directory needs to be writable by group (g+w).
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
apiVersion: rbac.authorization.k8s.io/v1  # Le ClusterRoles plus bas "filebeat" sera à présent associés à un compte et ce compte c'est le ServiceAccount filebeat
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1   # Le roles plus bas "filebeat" sera à présent associés à un compte et ce compte c'est le ServiceAccount filebeat
kind: RoleBinding
metadata:
  name: filebeat
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: kube-system
roleRef:
  kind: Role
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1   # Le roles plus bas "filebeat-kubeadm-config" sera à présent associé à un compte et ce compte c'est le ServiceAccount filebeat
kind: RoleBinding
metadata:
  name: filebeat-kubeadm-config
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: kube-system
roleRef:
  kind: Role
  name: filebeat-kubeadm-config
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1    # Definition d'un ClusterRole. La particularité c'est qu'un clusterrole n'est pas limité à un namespace
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
- apiGroups: ["apps"]
  resources:
    - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources:
    - jobs
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1        # Voir L'explication du role suivant 
kind: Role
metadata:
  name: filebeat
  # should be the namespace where filebeat is running
  namespace: kube-system
  labels:
    k8s-app: filebeat
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs: ["get", "create", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1        # La définition d'un Role. qui sera créer dans le namespace kube-system. Juste pour rappel, un Role a pour scope namespace, en dehors de ce namespace ce compte n'a pas de droits
kind: Role
metadata:
  name: filebeat-kubeadm-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
rules:                                      # La règle de ce Role c'est qu'il donne accès à la ressource du nom de kubeadm-config le Pouvoir de Lecture (GET)
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - kubeadm-config
    verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount               # Aussi simple que ça, on crée un compte de service ( service account) auquel on donnera les droit Role et des droits Cluster (ClusterRole)
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
---
```

</details>