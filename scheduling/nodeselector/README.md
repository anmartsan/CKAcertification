# NODE SELECTOR

Etiquetamos los nodos 
~~~
kubectl label node k8s-worker1 size=large
~~~

Creamos el Pod con el parametro nodeSelector
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
  nodeSelector:
    size: large
~~~ 
