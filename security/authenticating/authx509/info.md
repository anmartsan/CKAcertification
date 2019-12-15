# Autenticacion mediante x509

## Segun la documentacion de Kubernetes:

>All Kubernetes clusters have two categories of users: service accounts managed by Kubernetes, and normal users.

>Normal users are assumed to be managed by an outside, independent service. An admin distributing private keys, a user store like Keystone or Google Accounts, even a file with a list of usernames and passwords. In this regard, Kubernetes does not have objects which represent normal user accounts. Normal users cannot be added to a cluster through an API call.

>In contrast, service accounts are users managed by the Kubernetes API. They are bound to specific namespaces, and created automatically by the API server or manually through API calls. Service accounts are tied to a set of credentials stored as Secrets, which are mounted into pods allowing in-cluster processes to talk to the Kubernetes API.**

## Autenticacion mediante x509 para usuarios normales

Vamos a crear un usuario 
Username: antonio
Group: scm

Vamos a añadir las politicas RBAC necesarias para que un usuario pueda manejar sus deployments en el namespace "developer"


### 1: Create The Office Namespace

kubectl create namespace developer

### 2: Create The User Credentials

 Kubernetes no tiene API para "User Accounts" . Uno de los metodos para manejar la authorizacion son los certificados.

Creamos la llave primaria

>openssl genrsa -out antonio.key 2048

Creamos el CSR , con CN como usuario y O como grupo.

>openssl req -new -key antonio.key -out antonio.csr -subj "/CN=antonio/O=scm"

Utilizaremos el Certificate authority "CA" y su llave privada  de kubernetes. Suelen estar en /etc/kubernetes/pki


>openssl x509 -req -in antonio.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out antonio.crt -days 500

Tanto el fichero crt y key deberian se guardados en algun lugar seguro. Esto al ser una prueba no es necesario.

Añadimos las credenciales al cluster kubernetes.


>kubectl config set-credentials antonio --client-certificate=antonio.crt  --client-key=antonio.key
>kubectl config set-context antonio-context --cluster=kubernetes  --namespace=developer --user=antonio

Una vez creado el usuario le tenemos que dar los permisos RBAC necesarios.


### 3: Create The Role For Managing Deployments


~~~
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: developer
  name: developer-admin
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] 
~~~
  

> kubectl create -f developer-admin.yaml

### 4: Bind el  role a antonio role

~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: developer-admin-binding
  namespace: developer
subjects:
- kind: User
  name: antonio
  apiGroup: ""
roleRef:
  kind: Role
  name: deployment-admin
  apiGroup: ""
  ~~~

> kubectl create -f rolebinding-developer-admin.yaml

### 5: Test The RBAC Rule


> kubectl --context=antonio-context run --image busybox
> kubectl --context=antonio-context get pods


> kubectl --context=antonio-context get pods --namespace=default

