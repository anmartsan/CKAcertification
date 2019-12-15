Autenticacion mediante x509

Segun la documentacion de Kubernetes:

All Kubernetes clusters have two categories of users: service accounts managed by Kubernetes, and normal users.

Normal users are assumed to be managed by an outside, independent service. An admin distributing private keys, a user store like Keystone or Google Accounts, even a file with a list of usernames and passwords. In this regard, Kubernetes does not have objects which represent normal user accounts. Normal users cannot be added to a cluster through an API call.

In contrast, service accounts are users managed by the Kubernetes API. They are bound to specific namespaces, and created automatically by the API server or manually through API calls. Service accounts are tied to a set of credentials stored as Secrets, which are mounted into pods allowing in-cluster processes to talk to the Kubernetes API.

Use Case 1: Create User With Limited Namespace Access
In this example, we will create the following User Account:

Username: employee
Group: bitnami
We will add the necessary RBAC policies so this user can fully manage deployments (i.e. use kubectl run command) only inside the office namespace. At the end, we will test the policies to make sure they work as expected.

Step 1: Create The Office Namespace
Execute the kubectl create command to create the namespace (as the admin user):

kubectl create namespace office
Step 2: Create The User Credentials
As previously mentioned, Kubernetes does not have API Objects for User Accounts. Of the available ways to manage authentication (see Kubernetes official documentation for a complete list), we will use OpenSSL certificates for their simplicity. The necessary steps are:

Create a private key for your user. In this example, we will name the file employee.key:

openssl genrsa -out employee.key 2048
Create a certificate sign request employee.csr using the private key you just created (employee.key in this example). Make sure you specify your username and group in the -subj section (CN is for the username and O for the group). As previously mentioned, we will use employee as the name and bitnami as the group:

openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=bitnami"
Locate your Kubernetes cluster certificate authority (CA). This will be responsible for approving the request and generating the necessary certificate to access the cluster API. Its location is normally /etc/kubernetes/pki/. In the case of Minikube, it would be ~/.minikube/. Check that the files ca.crt and ca.key exist in the location.

Generate the final certificate employee.crt by approving the certificate sign request, employee.csr, you made earlier. Make sure you substitute the CA_LOCATION placeholder with the location of your cluster CA. In this example, the certificate will be valid for 500 days:

openssl x509 -req -in employee.csr -CA CA_LOCATION/ca.crt -CAkey CA_LOCATION/ca.key -CAcreateserial -out employee.crt -days 500
Save both employee.crt and employee.key in a safe location (in this example we will use /home/employee/.certs/).

Add a new context with the new credentials for your Kubernetes cluster. This example is for a Minikube cluster but it should be similar for others:

kubectl config set-credentials employee --client-certificate=/home/employee/.certs/employee.crt  --client-key=/home/employee/.certs/employee.key
kubectl config set-context employee-context --cluster=minikube --namespace=office --user=employee
Now you should get an access denied error when using the kubectl CLI with this configuration file. This is expected as we have not defined any permitted operations for this user.

kubectl --context=employee-context get pods
Step 3: Create The Role For Managing Deployments
Create a role-deployment-manager.yaml file with the content below. In this yaml file we are creating the rule that allows a user to execute several operations on Deployments, Pods and ReplicaSets (necessary for creating a Deployment), which belong to the core (expressed by “” in the yaml file), apps, and extensions API Groups:

kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: office
  name: deployment-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # You can also use ["*"]
Create the Role in the cluster using the kubectl create role command:

kubectl create -f role-deployment-manager.yaml
Step 4: Bind The Role To The Employee User
Create a rolebinding-deployment-manager.yaml file with the content below. In this file, we are binding the deployment-manager Role to the User Account employee inside the office namespace:

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: deployment-manager-binding
  namespace: office
subjects:
- kind: User
  name: employee
  apiGroup: ""
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: ""
Deploy the RoleBinding by running the kubectl create command:

kubectl create -f rolebinding-deployment-manager.yaml
Step 5: Test The RBAC Rule
Now you should be able to execute the following commands without any issues:

kubectl --context=employee-context run --image bitnami/dokuwiki mydokuwiki
kubectl --context=employee-context get pods
If you run the same command with the --namespace=default argument, it will fail, as the employee user does not have access to this namespace.

kubectl --context=employee-context get pods --namespace=default
Now you have created a user with limited permissions in your cluster.
