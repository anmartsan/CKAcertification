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



## Autenticacion mediante CertificateSigningRequest


Creamos la llave primaria

> openssl genrsa -out antonio.key 2048

Creamos el request.

> openssl req -new -key antonio.key -subj "/CN=antonio" -out antonio.csr

Lo convertimos a base64

> cat antonio.csr |base64

Creamos el objeto CertificateSigningRequest


~~~
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: antonio
spec:
  request:
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1Z6Q0NBVDhDQVFBd0VqRVFNQTRHQTFVRUF3d0hZVzUwYjI1cGJ6Q0NBU0l3RFFZSktvWklodmNOQVFFQgpCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFOSXllSjdKMWE5Y3RoV2xtNmlkb2h5Mk1BRklqcGF2b3FZSGRvakdOb0hUCit0RlJockZyVmdoSWM2ZkFXMFJDMU4zSldQaHM5R2Vubm9WQzBYaGE4NXVUeGJUWFRDMU9XcTJQYUFEUU1nM1kKUG1oNTVNN2M5NDVZT3plNWRHNllCNDR4NzhLVUVBZmdjc2E2YW05Y2h4bjdkNnJ6czZ3NUFreDhJclFVQmwwUQo4L0pOQ09kd0tDZjBmeWpoZEYzTkhPUmJnSFFYQ3AwMmtsVTU2NzlWZDU1bmFkTmhWalRUUFQ5SjBOWFBkV1hiCnBxSUN5dW9haDN3M09yNWtPUUpzMTV5ZU1STFpwaVB4S1ZBbENCNEtWcjdXazdNclM4c1FxZW1mZkJqU2lodkgKM1hqaVhHVWFGVWUwcnRZUmhKOFFSVE1iQnpwbnlHMUw0a1ZCZTczYXlVVUNBd0VBQWFBQU1BMEdDU3FHU0liMwpEUUVCQ3dVQUE0SUJBUUNnR2RkNm9wTEZoWEhaZ1Vrb3VyMWRnLzFpNGQwWVFhZU5aS1pQUmJybkdCSUZ2SFF3CndYRzZSSXlVSVdNTnk3Um1qYkFqbThNU0RxNVFvcjNmVU5oUTJHdnM1MUU2N244NUp6MVgzcUtLZWplQk5rQ2oKeXZoY3NYMWJ0VGI1U0F2OENvL0dOMm54a3dZdFhCRG4zTC9PdjU1RGtZRXU0R3pwTDF3MkJlRW0rNXg0UzFUagpqaCtDWGwyVmtIQU9YdjRucXFkRjBybzlPY1k0RFVMTGl6bERkSmRaSTBtVWQ1Qk8vcWZ5UVhrazZUeWYyYkJ1Cld2SXdjL2RlZGVTSDBydFJBS1dvQ0QwNzA0VlJ4Q3M0S2tvbndrR1V4VXRjVDBnZHlxWmNYL1h5a2dmQjFtZTgKWHB6RTFEakhQWDl0SXN5MFVjYUxjTUZmdVhES1BQYUVLWHZKCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=

  usages:
  - digital signature
  - key encipherment
  - server auth

~~~

Insertamos el certificado y aceptamos el certificado.


kubeadm create -f csr.yaml
kubeadm certificate approve antonio


Ya podemos compartir el certificado:

k get csr antonio -o yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  creationTimestamp: "2019-12-23T14:24:13Z"
  name: antonio
  resourceVersion: "1467567"
  selfLink: /apis/certificates.k8s.io/v1beta1/certificatesigningrequests/antonio
  uid: f5b04cb0-f1ea-49fe-be91-91e64fb2a7af
spec:
  groups:
  - system:masters
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1Z6Q0NBVDhDQVFBd0VqRVFNQTRHQTFVRUF3d0hZVzUwYjI1cGJ6Q0NBU0l3RFFZSktvWklodmNOQVFFQgpCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFOSXllSjdKMWE5Y3RoV2xtNmlkb2h5Mk1BRklqcGF2b3FZSGRvakdOb0hUCit0RlJockZyVmdoSWM2ZkFXMFJDMU4zSldQaHM5R2Vubm9WQzBYaGE4NXVUeGJUWFRDMU9XcTJQYUFEUU1nM1kKUG1oNTVNN2M5NDVZT3plNWRHNllCNDR4NzhLVUVBZmdjc2E2YW05Y2h4bjdkNnJ6czZ3NUFreDhJclFVQmwwUQo4L0pOQ09kd0tDZjBmeWpoZEYzTkhPUmJnSFFYQ3AwMmtsVTU2NzlWZDU1bmFkTmhWalRUUFQ5SjBOWFBkV1hiCnBxSUN5dW9haDN3M09yNWtPUUpzMTV5ZU1STFpwaVB4S1ZBbENCNEtWcjdXazdNclM4c1FxZW1mZkJqU2lodkgKM1hqaVhHVWFGVWUwcnRZUmhKOFFSVE1iQnpwbnlHMUw0a1ZCZTczYXlVVUNBd0VBQWFBQU1BMEdDU3FHU0liMwpEUUVCQ3dVQUE0SUJBUUNnR2RkNm9wTEZoWEhaZ1Vrb3VyMWRnLzFpNGQwWVFhZU5aS1pQUmJybkdCSUZ2SFF3CndYRzZSSXlVSVdNTnk3Um1qYkFqbThNU0RxNVFvcjNmVU5oUTJHdnM1MUU2N244NUp6MVgzcUtLZWplQk5rQ2oKeXZoY3NYMWJ0VGI1U0F2OENvL0dOMm54a3dZdFhCRG4zTC9PdjU1RGtZRXU0R3pwTDF3MkJlRW0rNXg0UzFUagpqaCtDWGwyVmtIQU9YdjRucXFkRjBybzlPY1k0RFVMTGl6bERkSmRaSTBtVWQ1Qk8vcWZ5UVhrazZUeWYyYkJ1Cld2SXdjL2RlZGVTSDBydFJBS1dvQ0QwNzA0VlJ4Q3M0S2tvbndrR1V4VXRjVDBnZHlxWmNYL1h5a2dmQjFtZTgKWHB6RTFEakhQWDl0SXN5MFVjYUxjTUZmdVhES1BQYUVLWHZKCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  usages:
  - digital signature
  - key encipherment
  - server auth
  username: kubernetes-admin
status:
  certificate: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1akNDQWM2Z0F3SUJBZ0lRYkMzVjhQTmRMUkh5K1RqOWdOV3RLakFOQmdrcWhraUc5dzBCQVFzRkFEQVYKTVJNd0VRWURWUVFERXdwcmRXSmxjbTVsZEdWek1CNFhEVEU1TVRJeU16RTBNakF5TUZvWERUSXdNVEl5TWpFMApNakF5TUZvd0VqRVFNQTRHQTFVRUF4TUhZVzUwYjI1cGJ6Q0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQCkFEQ0NBUW9DZ2dFQkFOSXllSjdKMWE5Y3RoV2xtNmlkb2h5Mk1BRklqcGF2b3FZSGRvakdOb0hUK3RGUmhyRnIKVmdoSWM2ZkFXMFJDMU4zSldQaHM5R2Vubm9WQzBYaGE4NXVUeGJUWFRDMU9XcTJQYUFEUU1nM1lQbWg1NU03Ywo5NDVZT3plNWRHNllCNDR4NzhLVUVBZmdjc2E2YW05Y2h4bjdkNnJ6czZ3NUFreDhJclFVQmwwUTgvSk5DT2R3CktDZjBmeWpoZEYzTkhPUmJnSFFYQ3AwMmtsVTU2NzlWZDU1bmFkTmhWalRUUFQ5SjBOWFBkV1hicHFJQ3l1b2EKaDN3M09yNWtPUUpzMTV5ZU1STFpwaVB4S1ZBbENCNEtWcjdXazdNclM4c1FxZW1mZkJqU2lodkgzWGppWEdVYQpGVWUwcnRZUmhKOFFSVE1iQnpwbnlHMUw0a1ZCZTczYXlVVUNBd0VBQWFNMU1ETXdEZ1lEVlIwUEFRSC9CQVFECkFnV2dNQk1HQTFVZEpRUU1NQW9HQ0NzR0FRVUZCd01CTUF3R0ExVWRFd0VCL3dRQ01BQXdEUVlKS29aSWh2Y04KQVFFTEJRQURnZ0VCQUljeW4xZW5tVDlHNFg4VHN1YjlvcFFvSjJndDJCakhMV2JySG1Qb0M3UU5YK3Z3bTRvSgpxNXpndUhTN0o0cXArQzl4WTBoU01yTStJNkRFM3JvWS9KYkN4ZzhyTWtmdGlHUDVnRU1OK3dWeVFQVjIvRkpYCmw1eCtId3dpWUtTZlR3Wm5saTZ4ZVZFY0c5WHBqZzh1MDZYa01SS3pwMkRUdUluTURIcmFWZDVTVDVKcm4wYysKL1A4TWRXNnp3RlJEQlYwbHZ5TjZNOTB2SXEwWU1rYnBtUitrenVVeUVMcUVSdXJRUTNPYUMySUppSDgyek9SNgpLTVhDbThkSVlXYWN3U3FtOFNuK1dqOEJzK3pHRWZmVDhRTFc5b1M5eURSbGwwT3FpMzZpUUZxMkhUYURzRUpBCkxMbFIwRnFray9aYmV1ak9PRUJhRFJJTEMyOUd1M1JiSE1zPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  conditions:
  - lastUpdateTime: "2019-12-23T14:25:20Z"
    message: This CSR was approved by kubectl certificate approve.
    reason: KubectlApprove
    type: Approved
root@k8s-master:~/CKAcertification/security/authenticating/authx509# echo "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1akNDQWM2Z0F3SUJBZ0lRYkMzVjhQTmRMUkh5K1RqOWdOV3RLakFOQmdrcWhraUc5dzBCQVFzRkFEQVYKTVJNd0VRWURWUVFERXdwcmRXSmxjbTVsZEdWek1CNFhEVEU1TVRJeU16RTBNakF5TUZvWERUSXdNVEl5TWpFMApNakF5TUZvd0VqRVFNQTRHQTFVRUF4TUhZVzUwYjI1cGJ6Q0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQCkFEQ0NBUW9DZ2dFQkFOSXllSjdKMWE5Y3RoV2xtNmlkb2h5Mk1BRklqcGF2b3FZSGRvakdOb0hUK3RGUmhyRnIKVmdoSWM2ZkFXMFJDMU4zSldQaHM5R2Vubm9WQzBYaGE4NXVUeGJUWFRDMU9XcTJQYUFEUU1nM1lQbWg1NU03Ywo5NDVZT3plNWRHNllCNDR4NzhLVUVBZmdjc2E2YW05Y2h4bjdkNnJ6czZ3NUFreDhJclFVQmwwUTgvSk5DT2R3CktDZjBmeWpoZEYzTkhPUmJnSFFYQ3AwMmtsVTU2NzlWZDU1bmFkTmhWalRUUFQ5SjBOWFBkV1hicHFJQ3l1b2EKaDN3M09yNWtPUUpzMTV5ZU1STFpwaVB4S1ZBbENCNEtWcjdXazdNclM4c1FxZW1mZkJqU2lodkgzWGppWEdVYQpGVWUwcnRZUmhKOFFSVE1iQnpwbnlHMUw0a1ZCZTczYXlVVUNBd0VBQWFNMU1ETXdEZ1lEVlIwUEFRSC9CQVFECkFnV2dNQk1HQTFVZEpRUU1NQW9HQ0NzR0FRVUZCd01CTUF3R0ExVWRFd0VCL3dRQ01BQXdEUVlKS29aSWh2Y04KQVFFTEJRQURnZ0VCQUljeW4xZW5tVDlHNFg4VHN1YjlvcFFvSjJndDJCakhMV2JySG1Qb0M3UU5YK3Z3bTRvSgpxNXpndUhTN0o0cXArQzl4WTBoU01yTStJNkRFM3JvWS9KYkN4ZzhyTWtmdGlHUDVnRU1OK3dWeVFQVjIvRkpYCmw1eCtId3dpWUtTZlR3Wm5saTZ4ZVZFY0c5WHBqZzh1MDZYa01SS3pwMkRUdUluTURIcmFWZDVTVDVKcm4wYysKL1A4TWRXNnp3RlJEQlYwbHZ5TjZNOTB2SXEwWU1rYnBtUitrenVVeUVMcUVSdXJRUTNPYUMySUppSDgyek9SNgpLTVhDbThkSVlXYWN3U3FtOFNuK1dqOEJzK3pHRWZmVDhRTFc5b1M5eURSbGwwT3FpMzZpUUZxMkhUYURzRUpBCkxMbFIwRnFray9aYmV1ak9PRUJhRFJJTEMyOUd1M1JiSE1zPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==" |base64 -d
-----BEGIN CERTIFICATE-----
MIIC5jCCAc6gAwIBAgIQbC3V8PNdLRHy+Tj9gNWtKjANBgkqhkiG9w0BAQsFADAV
MRMwEQYDVQQDEwprdWJlcm5ldGVzMB4XDTE5MTIyMzE0MjAyMFoXDTIwMTIyMjE0
MjAyMFowEjEQMA4GA1UEAxMHYW50b25pbzCCASIwDQYJKoZIhvcNAQEBBQADggEP
ADCCAQoCggEBANIyeJ7J1a9cthWlm6idohy2MAFIjpavoqYHdojGNoHT+tFRhrFr
VghIc6fAW0RC1N3JWPhs9GennoVC0Xha85uTxbTXTC1OWq2PaADQMg3YPmh55M7c
945YOze5dG6YB44x78KUEAfgcsa6am9chxn7d6rzs6w5Akx8IrQUBl0Q8/JNCOdw
KCf0fyjhdF3NHORbgHQXCp02klU5679Vd55nadNhVjTTPT9J0NXPdWXbpqICyuoa
h3w3Or5kOQJs15yeMRLZpiPxKVAlCB4KVr7Wk7MrS8sQqemffBjSihvH3XjiXGUa
FUe0rtYRhJ8QRTMbBzpnyG1L4kVBe73ayUUCAwEAAaM1MDMwDgYDVR0PAQH/BAQD
AgWgMBMGA1UdJQQMMAoGCCsGAQUFBwMBMAwGA1UdEwEB/wQCMAAwDQYJKoZIhvcN
AQELBQADggEBAIcyn1enmT9G4X8Tsub9opQoJ2gt2BjHLWbrHmPoC7QNX+vwm4oJ
q5zguHS7J4qp+C9xY0hSMrM+I6DE3roY/JbCxg8rMkftiGP5gEMN+wVyQPV2/FJX
l5x+HwwiYKSfTwZnli6xeVEcG9Xpjg8u06XkMRKzp2DTuInMDHraVd5ST5Jrn0c+
/P8MdW6zwFRDBV0lvyN6M90vIq0YMkbpmR+kzuUyELqERurQQ3OaC2IJiH82zOR6
KMXCm8dIYWacwSqm8Sn+Wj8Bs+zGEffT8QLW9oS9yDRll0Oqi36iQFq2HTaDsEJA
LLlR0Fqkk/ZbeujOOEBaDRILC29Gu3RbHMs=
-----END CERTIFICATE-----

