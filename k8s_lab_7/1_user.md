# Création d'utilisateur sur Kubernetes:

Dans Kubernetes, il n'y a pas d'objets "utilisateurs" ni de registre d'utilisateurs.
L'autheification des requêtes se fait soit :

* Static token
* Bearer token
* Certificat X509
* OIDC
* etc

Il est préférable d'utiliser une sorte de fournisseur OpenId Connect.

Dans ce la, nous utilisons une authentification basée sur les certificats, et le "subjet" du certificat est considéré comment le "nom d'utilisateur" dans Kubernetes.

Les grandes lignes du processus sont :

* créer une clé privée RSA
* créer une demande de signature de certificat (CSR) pour cette clé privée
* envoyer cette CSR à Kubernetes et la signer avec son autorité de certification
* récupérer le certificat signé
* créer un fichier de configuration pour kubectl
* vérifier que nous pouvons parler à Kubernetes mais que nous n'avons pas de privilèges tant que les Rolebinding et ClusterRolebing ne sont pas définis
* accorder les privilèges

Créer l'utilisteur **devuser**

Générez à l'aide de openssl une clé privée (PKCS1, 2048, PEM)

```bash
openssl genrsa -out devuser.pem
```

créer une demande de signature CSR (qui sera utilisée à l'étape suivante)

* `CN` : Il s'agit du nom d'utilisateur
* `O` : Nom de l'organisation. Il est en fait utilisé comme groupe par Kubernetes lors de l'authentification/autorisation des utilisateurs. Vous pouvez en ajouter autant que vous le souhaitez

```bash
openssl req -new -key devuser.pem -out devuser.csr -subj "/CN=devuser/O=dev/O=wizetraining"
```

Transmettre la demande de signature à kubernetes pour cela :

* Convertissez le CSR en base64 : 

```bash
cat devuser.csr | base64
```

* Créer un objet K8s `CertificateSigningRequest`

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: devuser
spec:
  request: <##### coller le Contenu Base64#####>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 3600
  usages:
  - client auth

```

```bash
kubectl apply -f devuser-csr.yml

kubectl get csr

kubectl get csr devuser

kubectl describe csr devuser

(Condition pending, nous devons l'approuver)
```

Signer la requête CSR

```bash
kubectl certificate approve devuser

kubectl get csr devuser

(Condition: Approved,Issued => nous l'avons approuvé. Il est signé)
```

Nous allons extraire le certificat signé

```bash
kubectl get csr devuser -o jsonpath='{.status.certificate}' | base64 -d > devuser.crt
```

<details><summary>Une autre alternative</summary>

Une autre alternative serait de signer le certificat avec openssl sur le Master node connaissant où se trouve la clé privée du CA

```bash
openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -days 10 -in devuser.csr -out devuser.crt
```

</details>


Nous pouvons désormais créer un contexte utilisateur pour cet utilisateur.
créer un fichier de configuration kubeconfig : récupérer le endpoint de l'api kubernetes, récupérer le bundle CA.

```bash
kubectl config set-cluster k8s-lab --insecure-skip-tls-verify=true --server=https://10.0.0.10:6443

kubectl config set-credentials devuser --client-certificate=devuser.crt --client-key=devuser.pem --embed-certs=true

kubectl config set-context default --cluster=k8s-lab --user=devuser

kubectl config use-context default

kubectl config view
```

nous n'avons plus besoin de csr, les fichiers temporaires ont été intégrés dans la configuration

```bash
kubectl delete csr devuser
rm devuser.pem devuser.crt devuser.csr
```

Avec le nouvel utilisateur, lister les Pods.
pourquoi ça ne marche pas ?

```bash
kubectl get pods
```


En vérifiant on constate que nous pouvons intéroger notre cluster kubernetes mais que nous n'avons pas de privilèges. 
Message d'erreur :

```error
L'utilisateur "devuser" ne peut pas lister la ressource "pods" dans le groupe API "" à l'échelle du cluster.
``` 

Certe l'utilisateur est authentifié mais il n'a pas les authorisations

_(Dans le Lab suivant nous lui donnerons les droits)_

une commande utile pour check les privilèges d'utilisateur :

```bash
kubectl auth can-i list pods --as devuser
```

### Notes

Il est possible d'utiliser `expirationSeconds` pour limiter la durée de validité du certificat. (voir la création CSR)

Lorsque nous créons le fichier de configuration `config`, nous demandons à kubectl d'intégrer des données de certificats, donc dans config :

* **client-key-data** - contenu encodé en base64 de notre fichier de clé privée rsa
* **client-certificate-data** - contenu encodé en base64 du certificat signé que nous avons récupéré de Kubernetes.

Notez qu'il n'y a aucun moyen de révoquer, le certificat signé sera valide pour toujours, mais nous pouvons toujours supprimer les Roles, les ClusterRoles de cluster de sorte que l'utilisateur certe puisse s'authentifier mais n'a pas de privilèges pour faire quoi que ce soit ni voir quoi que ce soit.
