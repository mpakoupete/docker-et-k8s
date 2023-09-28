
# Role Base Access Control

Quelle est la différence entre Roles et ClusterRoles ?

<details><summary>Correction</summary>

* Le rôle est limité au namespace (default/K8s-lab/dev)
* ClusterRole est global dans le Cluster, donc englobe aussi les namespaces

</details>

Pour le namespace spécifique au projet de developpement `dev-project` (à créer),vous souhaitiez fournir un accès en lecture seule sur toutes les ressources K8s aux utilisateurs du groupe `dev` ; créer le Role correspondant.

<details><summary>Correction</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-project
  name: readonly-for-dev
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

</details>

Créer le Rolebinding correspondant et faites les tests

<details><summary>Correction</summary>

```bash
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-for-dev
  namespace: dev-project
subjects:
- kind: Group
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly-for-dev
  apiGroup: rbac.authorization.k8s.io
```

</details>

Tester :

```bash
kubectl auth can-i list pods --namespace dev-project --as devuser
yes

kubectl auth can-i list pods --namespace default --as devuser
no

kubectl auth can-i create pods --namespace dev-project --as devuser
no
```

Créer un Role et un Rolebinding pour permettre à l'utilisateur devuser uniquement de : 
* créer des ressources : Pod, Deployment => dans le Namespace default
* lister les ressources : configmap, services => dans le Namespace default