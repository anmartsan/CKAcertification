#Static Pods - No Scheduled Pods

Nos conectamos al nodo donde queremos ejecutar el pod estatico.

~~~
ps -ef |grep kubelet
root      1390     1  2 Dec13 ?        01:39:05 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1
~~~

Editamos el config.yaml de kubelet (/var/lib/kubelet/config.yaml)

Añadimos si no esta configurado :
staticPodPath: /etc/kubernetes/manifests

Reiniciamos kubelet:

~~~
systemctl restart kubelet
~~~

Añadimos un pod a ese directorio.

~~~
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP

kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
static-web-k8s-worker1   1/1     Running   0          8s
~~~
