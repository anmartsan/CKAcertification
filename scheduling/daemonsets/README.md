~~~
kubectl create deploy nginx --image=nginx --dry-run -o yaml > ds.yaml
~~~

Una vez que tengamos el fichero yaml, modificamos Deployment por DaemonSet y quitamos las replicas.


