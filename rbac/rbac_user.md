**Create cert with O which is NOT system:masters**
```bash
cat > operator-csr.json <<EOF
{
  "CN": "operator",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "BGLR",
      "O": "system:view",
      "OU": "CKA"
    }
  ]
}
EOF


cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  operator-csr.json | cfssljson -bare operator
```

**kubectl commands to create user along with rolebindings**
```bash
SERVER=$(kubectl config view -o jsonpath='{.clusters[?(@.name=="kubernetes")].cluster.server}')
kubectl config set-cluster operator --certificate-authority ca.pem --server $SERVER
kubectl config set-credentials operator --client-certificate operator.pem --client-key operator-key.pem
kubectl config set-context operator --cluster operator --user operator
kubectl auth can-i get pods --as operator
kubectl get roles
kubectl get clusterroles
kubectl describe clusterrole view
kubectl describe clusterrole admin
kubectl describe clusterrole cluster-admin
kubectl auth can-i "*" "*"
kubectl describe clusterrole edit
kubectl create rolebinding operator --clusterrole view --user operator -n default --save-config
kubectl create -f auth/crb-view.yml --record --save-config
kubectl auth can-i get pods --as operator --all-namespaces
kubectl config use-context operator
kubectl auth can-i "*" "*"

$ cat auth/crb-view.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view
subjects:
- kind: User
  name: operator
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```
