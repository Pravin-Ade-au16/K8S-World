### RABC: How to add user to k8s cluster.

1. Understanding KUBECONFIG
2. k8s Authentication
3. Create user in Linux
4. create certificate for users.
5. Configure your k8s cluster with kube config file
6. share users kubeconfig to the users home directory or any other dir.
7. create RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding and bind user to it.

**1. Understanding KUBECONFIG**
it's important to undestand it's three top level structures.

**Users:** By which name he/she will authenticate to a cluster

**Cluster:** cluster attribute provides all the data neccessary to connect to a cluster.
            **E.g.** CA Certificate, Self Sign certificate, etc.

**Context:** context is where we associate users with clusters as a single name entity.

**2. K8S Authentication:** 
k8s support a few different authentications provides.

X509 client certificate
Static token files on the host
Azure Active Directory and AWS IAM
Authentication webhooks

Here, I used to implement using x509 cert.

**3. Create user in Linux (Ubuntu)**.
useradd -m -d /home/<usrname> <user-name>
passwd <username>

**4. Create certificates for users**
create a key and certificate signing request (CSR) for users access to the cluster using OpenSSL

openssl req -new -newkey rsa:4096 -nodes -keyout <username>.key -out <username>.csr -subj "/CN=<usename>/O=adept"

Now, we have CSR, we need to have it signed by cluster CA. For that, we create a **CertificateSigningRequest** objeect within k8s containing the CSR we generate above.

**5. Create Certificate Signing Request**
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: <userName>-access # change as per your requirement
spec:
  groups:
  - system: authenticated
  request: $(cat <username.csr> | base64 -w 0)
signerName: kubernetes.io/kube-apiserver-client
usages:
- client auth
EOF
```
``` kubectl get csr ```

Now, CSR has been created, it enters a **PENDING** state.

``` kubectl get csr <username-access>```

Next, we need to approve the CSR object.

```
kubectl certificate approve <username-access> 
kubectl get csr <username-access>
```
users CSR has been made available in **status.certificate** file.
Next, need to retrieve the certificate

```
openssl x509 -req -in <username>.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out <username>.crtc -days 365
```
verify CSR

```
ls -ltr
cat <username>.crt
```
Next, requerment for users KUBECONFIG file.

```
kubectl config view -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' --raw | base64 --decode - > k8s-ca.crt
```

Note: do not change anything in above command run as it is. so, you get k8s certificate k8s-ca.crt

**6. Configure your k8s cluster with users kube config file**

Now, we can start creating users kubeconfig file.

```
kubectl config set-cluster $(kubectl config view -o jsonpath='{.cluster[0].name}') --server=$(kubectl config view -o jsonpath='{.cluster[0].cluster.server}') --certificate-authority=k8s-ca.crt --kubeconfig=<username>-config --embed-certs
```

setup the users next which will import users key and certificate into the file.

```
kubectl config set-credentials <usersname> --client-certificate=<username>.crt --client-key=<username.key> --embed-certs --kubeconfig=<username>-config
```

Now, users kubeconfig file ready to share with users home directory.

The final kubeconfig requirement is to create a context.

```
kubectl config set-context username --cluster=$(kubectl config view -o jsonpath='{.cluster[0].name}') --namespace=default --user=pravin --kubeconfig=username-config
```

Note: the --namespace parameter tell kubectl which namespace to use for that context, default in our case.
finaly, switch the context as user

``` kubectl config use-context username --kubeconfig=<username>-config ```

Now, let's test users kubeconfig by running the kubectl version command.

``` kubectl version --kubeconfig=<username>-config ```
We did not recieve any error shows that users kubecocnfig file is configured correctly.

**7. Shares users kube config to their home directory**
Next, we need to create /.kube directory in home directory of users and change ownership as well as execution permission to the users kubeconfig file so. user can run kubectl command without any error.

```
mkdir /home/user-name/.kube
chown user:user /home//user-name/.kube
chmod user:user /home//user-name/.kube
cp -i user-config /home/user-name/.kube/
chown user:user /home/user-name/.kube/user-config
chmod 777 /home/user-name/.kube/user-config
```

Now, switch user

Next, add environment variable permanently so, to do that
edit the ~/.profile and add - export KUBECONFIG=/home/user-name/.kube/user-config to end of the file. save and exit.


