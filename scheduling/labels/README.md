~~~
kubectl  run frontend --image=nginx --labels="tier=frontend" --restart=Never
kubectl  run backend --image=nginx --labels="tier=backend" --restart=Never
~~~ 
~~~ 
apiVersion: v1
kind: Pod
metadata:
  labels:
    tier: frontend
  name: app
spec:
  containers:
  - image: nginx
    name: app

apiVersion: v1
kind: Pod
metadata:
  labels:
    tier: backend
  name: web
spec:
  containers:
  - image: nginx
    name: web
~~~ 
~~~
kubectl get pod --show-labels


NAME       READY   STATUS    RESTARTS   AGE     LABELS
backend    1/1     Running   0          9m23s   tier=backend
frontend   1/1     Running   0          10m     tier=frontend
~~~
~~~
kubectl get pods --selector="tier=backend"
kubectl get pods --selector="tier=frontend"
kubectl get pods --selector="tier in (backend)"
~~~
