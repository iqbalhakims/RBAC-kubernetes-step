# Kubernetes User Creation via RBAC: `bob`

This repository provides a step-by-step guide to creating a Kubernetes user named `bob` with limited permissions using Role-Based Access Control (RBAC). It includes certificate generation, kubeconfig setup, and role assignments for `bob`.

## Prerequisites
- Access to a Kubernetes cluster with admin privileges.
- `kubectl` installed and configured.
- Access to Kubernetes CA files (`ca.crt` and `ca.key`).

---

## 1. Generate Certificates for `bob`

Kubernetes uses TLS client certificates for user authentication. Follow these steps to generate `bob`'s private key and certificate.

### Generate Private Key and CSR
Run the following commands to create a private key and a certificate signing request (CSR):

```bash
openssl genrsa -out bob.key 2048
openssl req -new -key bob.key -out bob.csr -subj "/CN=bob/O=developers"
```

### Sign the CSR with the Kubernetes CA
Use the Kubernetes CA to sign the CSR and generate the certificate:

```bash
openssl x509 -req -in bob.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out bob.crt -days 365
```

You should now have:
- `bob.key`: Private key
- `bob.crt`: Certificate signed by the Kubernetes CA

---

## 2. Add User Credentials to Kubeconfig

To allow `bob` to authenticate with the cluster, add their credentials to the kubeconfig file.

### Add `bob`'s Credentials
```bash
kubectl config set-credentials bob \
  --client-certificate=bob.crt \
  --client-key=bob.key
```

### Create a New Context for `bob`
```bash
kubectl config set-context bob-context \
  --cluster=<cluster-name> \
  --namespace=default \
  --user=bob
kubectl config use-context bob-context
```

> Replace `<cluster-name>` with the name of your Kubernetes cluster.

---

## 3. Create RBAC Role and RoleBinding

Define a Role and a RoleBinding to grant `bob` specific permissions within the `default` namespace.

### Role Definition (`role.yaml`)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "create", "delete"]
```

### RoleBinding Definition (`rolebinding.yaml`)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: default
  name: developer-binding
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### Apply the YAML Files
```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```

---

## 4. Test `bob`'s Access

Switch to `bob`'s context and verify access:

```bash
# Check access to resources
kubectl get pods --context=bob-context

# Try creating a deployment
kubectl create deployment nginx --image=nginx --context=bob-context
```

If everything is configured correctly, `bob` will have access to the resources allowed by the `developer-role`.

---

## Cleanup

To remove the configurations:

```bash
kubectl delete role developer-role -n default
kubectl delete rolebinding developer-binding -n default
kubectl config delete-context bob-context
kubectl config unset users.bob
rm bob.key bob.crt bob.csr
```

---

