> kubectl  run frontend --image=nginx --labels="tier=frontend" --restart=Never
> kubectl  run backend --image=nginx --labels="tier=backend" --restart=Never
> 
> kubectl get pod --show-labels
> 
> kubectl get pods --selector="tier=backend"
> kubectl get pods --selector="tier=frontend"
> kubectl get pods --selector="tier in (backend)"
