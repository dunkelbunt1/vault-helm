# This branch is there to show an error with Vault
```
Error initializing: Put https://127.0.0.1:8200/v1/sys/init: x509: certificate signed by unknown authority
```
Please follow the follwing steps to reproduce the error:

1. **Set the following enviroment variables in your shell**
```
export SERVICE=vault
export NAMESPACE=vault
export SECRET_NAME=vault-tls
export CSR_NAME=vault-csr
export BASENAME=vault
export TMPDIR=./cert
```

2. **Create a TMPdir**
```
mkdir ./cert
```
3. **Gernate the TLS key pair**
```
openssl genrsa -out ${TMPDIR}/${BASENAME}.key rsa 2048

cat <<EOF >${TMPDIR}/${BASENAME}-csr.conf
[ req ]\ndefault_bits = 2048\nprompt = no
encrypt_key = yes
default_md = sha256
distinguished_name = dn
req_extensions = v3_req
[ dn ]
C = UK
ST = London
L = London
O = Personal
emailAddress = youremail@domain.no
CN = ${SERVICE}.${NAMESPACE}.svc
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${SERVICE}
DNS.2 = ${SERVICE}.${NAMESPACE}
DNS.3 = ${SERVICE}.${NAMESPACE}.svc
DNS.4 = ${SERVICE}.${NAMESPACE}.svc.cluster.local
IP.1  = 127.0.0.1
EOF

openssl req -config ${TMPDIR}/${BASENAME}-csr.conf -new -key ${TMPDIR}/${BASENAME}.key -subj "/CN=${SERVICE}.${NAMESPACE}.svc" -out ${TMPDIR}/${BASENAME}.csr

 cat <<EOF >${TMPDIR}/${BASENAME}-csr.yaml
 apiVersion: certificates.k8s.io/v1beta1
 kind: CertificateSigningRequest
 metadata:
   name: ${CSR_NAME}\nspec:
     groups:
       - system:authenticated
         request: $(cat ${TMPDIR}/${BASENAME}.csr | base64 | tr -d '\n')
           usages:
             - digital signature
               - key encipherment
                 - server auth
  EOF
```

4. **Create a Namespace and upload your new Keypair**
```
kubectl create namespace vault
kubectl create -f ${TMPDIR}/${BASENAME}-csr.yaml --namespace ${NAMESPACE}
#Check if the secret is created
kubectl get csr ${CSR_NAME} --namespace ${NAMESPACE}
#If the status is on issued go ahead and approve it
kubectl certificate approve ${CSR_NAME} --namespace ${NAMESPACE}
serverCert=$(kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}')
echo "${serverCert}" | openssl base64 -d -A -out ${TMPDIR}/${BASENAME}.crt
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d > ${TMPDIR}/${BASENAME}.ca
kubectl create secret generic ${SECRET_NAME} --namespace ${NAMESPACE} --from-file=${BASENAME}.key=${TMPDIR}/${BASENAME}.key --from-file=${BASENAME}.crt=${TMPDIR}/${BASENAME}.crt --from-file=${BASENAME}.ca=${TMPDIR}/${BASENAME}.ca

```
5. **After you copied and pasted your way through it, it is time do deploy vault**
```
helm install vault  --set='server.ha.enabled=true' --set='server.ha.raft.enabled=true' .
```

6. **Watch your deployment**
```
kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          2m52s
vault-1                                 0/1     Pending   0          2m51s
vault-2                                 0/1     Pending   0          2m50s
vault-agent-injector-588476fbc7-thfwx   1/1     Running   0          2m52s
```
7. **Switch your namespace and try to unseal vault**
```
kubectl config set-context --current --namespace=vault
kubectl exec -ti vault-0 -- vault operator init
```
**Requirments:** 
```
helm v.3.2.x, kubectl version: 1.18.x, minikube version: v1.9.x
```