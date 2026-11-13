STEPS for granting Cluster Access to a new user (marco) using Certificates and RBAC to securely grant access to 
a kubernetes cluster within defined scope with certificate-based authentication and RBAC.

step 1: user marco generates a Private Key
openssl genrsa -out marco.key 2048    # This generate a 2048-bit RSA private key which MUST REMAIN PRIVATE AND SECURE.

step 2: marco generates a Certificate Signing Request (CSR) using marco.key Private Certificate
openssl req -new -key marco.key -out marco.csr -subj "/CN=marco"   # -subj "/CN=marco" sets the Common Name (CM) to marco.

step 3: marco shares the CSR  with the K8s Admin for Approval
cat marco.csr | base64 | tr -d "\n"    # this command converts the CSR into singe-line BASE64 strings, stripping newlines with tr -d "\n" which is necessary for YAML formatting.

step 4: Kubernetes Admin creates the CSR object in k8s

---
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: marco-csr
spec:
  #This is an encoded CSR. Chane this to base64-encoded contents of marco.csr
  request: <BASE64-ENCODED-CSR> #Put the output (content) of the previous command here. 
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 7776000  # 90 days in seconds
  usages:
    - client auth 

Apply the csr.yaml file 

step 5: Kubernetes Admin approves the CSR
kubectl certificate approve marco-csr  #marco-csr is the name from the CSR file, so replace it with yours.

step 6: K8s Admin retrieves and shares the Signed Certificate
kubectl get csr marco-csr -o jsonpath='{.status.certificate}' | base64 -d > marco.crt

The Admin retrieves the signed certificate from CSR's status, decodes it from base64, and saves it as marco.crt.
This certificate, along with marco.key, is sent back to Marco for kubeconfig configuration.

step 7: Marco configure his kubeconfig with credentials and cluste info.     

Set user marco credentials:
kubectl config set-credentials marco \
  --client-certificate=marco.crt \
  --client-key=marco.key \
  --certificate-authority=ca.crt \
  --embed-certs=true

Set Cluster server endpoint to use to correct CA (ca.crt): We already have an existing cluster that we are using
kubectl config set-cluster kind-my-first-cluster \
  --server=https://127.0.0.1:33325 \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --kubeconfig=~/.kube/config

Set a context for user marco in kind-my-first-cluster cluster and marco@kind-my-first-cluster-context
kubectl config set-context marco@kind-my-first-cluster-context \
  --cluster=kind-my-first-cluster \ 
  --user=marco \
  --namespace=default

These command set and configure user credentials, cluster endpoint, and context in Marco's kubeconfig.

step 8: Configure authorization for Marco user.
Create a role for defined scope you want marco to have.

vim marco-role.yaml and add following:  
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: marco-role
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - delete 

kubectl apply -f marco-role.yaml

And a rolebinding so that marco user can use the defined role.
kubectl create rolebinding marco-rbind \
  --role=marco-role \
  --user=marco \
  --namespace=default

step 9: As Adim verify authorization with can-i 
kubectl auth can-i delete pods --namespace --as=marco

This allows the Admin to verify whether the authorization and RBAC given to user marco are working in the default namespace.
