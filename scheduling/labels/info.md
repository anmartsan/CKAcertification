~~~
kubectl  run frontend --image=nginx --labels="tier=frontend" --restart=Never
kubectl  run backend --image=nginx --labels="tier=backend" --restart=Never
~~~ 
~~~
kubectl get pod --show-labels
~~~

NAME       READY   STATUS    RESTARTS   AGE     LABELS
backend    1/1     Running   0          9m23s   tier=backend
frontend   1/1     Running   0          10m     tier=frontend

kubectl get pods --selector="tier=backend"
kubectl get pods --selector="tier=frontend"
kubectl get pods --selector="tier in (backend)"
