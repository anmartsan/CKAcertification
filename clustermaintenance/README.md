apt update
apt-cache policy kubeadm

Miramos la version que queremos instalar.
1.15.0-00 500
        500 https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages

Actualizamos primero kubeadm.
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.15.7-00 && \
apt-mark hold kubeadm

kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:37:41Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}

kubeadm upgrade plan
kubeadm upgrade apply v1.15.7


Ahora el upgrade de kubelet y kubectl

apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.15.7-00 kubectl=1.15.7-00 && \
apt-mark hold kubelet kubectl


Actualizamos cada uno de los nodos uno por uno

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.15.7-00 && \
apt-mark hold kubeadm

kubectl drain $NODE --ignore-daemonsets
kubeadm upgrade node
 systemctl restart kubelet
kubectl uncordon
