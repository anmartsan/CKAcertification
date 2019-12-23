# NODE AFFINITY

Con node affinity podemos asignar el nodo en el que ejecutamos el POD de una manera mas precisa.

Con nodeSelector no podemos usar expresiones del tipo "Large o small" o no "!grande" etc. Nodeaffinity si nos permite usar expresiones.

~~~
affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: size
            operator: In
            values:
            - large
~~~

~~~
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: size
            operator: In
            values:
            - large
        topologyKey: failure-domain.beta.kubernetes.io/zone
  containers:
  - image: nginx
    name: nginx

~~~
