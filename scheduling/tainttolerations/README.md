# Taints y tolerations


>Un taint de nodo te permite marcar un nodo a fin de que el programador evite o impida su uso para ciertos pods. Una característica complementaria, las tolerancias, te permite designar pods que se pueden usar en nodos “tainted”.

>Los taints de nodo son pares clave-valor asociados con un efecto. Los efectos disponibles se muestran a continuación:

>NoSchedule: los pods que no toleran este taint no están programados en el nodo.
>PreferNoSchedule: Kubernetes evita la programación de pods que no toleran este taint en el nodo.
>NoExecute: el pod se desaloja del nodo si ya está en ejecución en este y no está programado en el nodo si aún no está en ejecución en él.

>A node that has a taint with

>"NoSchedule" effect will prevent PODs to be scheduled on this node, as long as the POD has not a matching toleration defined.
>The "NoExecute" effect will terminate any running PODs without matching tolerations on the node, while
>"PreferNoSchedule" is a soft requirement. A POD with no toleration will prefer to not start on a node with a „PreferNoSchedule“ taint unless it does not have another choice and would enter the „Pending“ status otherwise.

## Taints

~~~
kubectl taint nodes <node-name> key=value:taint-effect
~~~

Creamos un taint en el nodo k8s-worker1

~~~
kubectl taint node k8s-worker1  myKey=myValue:NoSchedule
kubectl describe node k8s-worker1 |grep Tain
Taints:             myKey=myValue:NoSchedule
~~~

Como podemos ver no se crean pods en el nodo k8s-worker1
~~~

kubectl run nginx --image=nginx --replicas=2 --restart=Always
kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
nginx-6db489d4b7-psscb   1/1     Running   0          12s   192.168.126.14   k8s-worker2   <none>           <none>
nginx-6db489d4b7-r99c4   1/1     Running   0          12s   192.168.126.13   k8s-worker2   <none>           <none>
nginx-x474l              1/1     Running   0          13h   192.168.126.12   k8s-worker2   <none>           <none>

~~~
Ahora quitamos el taint al nodo.

~~~
kubectl  taint node k8s-worker1 myKey-
kubectl scale --replicas=3 deployment/nginx
k get pods -o wide --
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
nginx-6db489d4b7-ckxm7   1/1     Running   0          4s      192.168.194.76   k8s-worker1   <none>           <none>
nginx-6db489d4b7-psscb   1/1     Running   0          7m16s   192.168.126.14   k8s-worker2   <none>           <none>
nginx-6db489d4b7-r99c4   1/1     Running   0          7m16s   192.168.126.13   k8s-worker2   <none>           <none>
nginx-x474l              1/1     Running   0          13h     192.168.126.12   k8s-worker2   <none>           <none>
~~~

## Tolerations

Creamos un taint en el nodo k8s-worker1

~~~
kubectl taint node k8s-worker1  myKey=myValue:NoSchedule
kubectl describe node k8s-worker1 |grep Tain
Taints:             myKey=myValue:NoSchedule
~~~

Creamos el deployment y le añadimos el apartado tolerations
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:
      - key: "myKey"
        operator: "Equal"
        value: "myValue"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
~~~

Los pods se ejecutaran en ambos nodos

~~~
kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
nginx-deployment-6f87f555c6-9f8ht   1/1     Running   0          69s   192.168.126.15   k8s-worker2   <none>           <none>
nginx-deployment-6f87f555c6-n8jxg   1/1     Running   0          69s   192.168.194.77   k8s-worker1   <none>           <none>

~~~
