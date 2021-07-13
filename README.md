# KIND CALICO

## Pre-Requisites

You will need to have the following softwares installed in order to run the cluster creation and calico installation:
- kind
- helm

## Create cluster

```bash
kind create cluster --name calico-cluster --config kind/kind-calico.yaml
```

## Install calico

```bash
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.19.1/calicoctl" 
sudo mv calicoctl /usr/local/bin
chmod +x /usr/local/bin/calicoctl


helm show values https://github.com/projectcalico/calico/releases/download/v3.18.4/tigera-operator-v3.18.4-1.tgz > calico/calico-values.yaml
helm install calico https://github.com/projectcalico/calico/releases/download/v3.18.4/tigera-operator-v3.18.4-1.tgz -f calico/calico-values.yaml --wait


kubectl -n calico-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
kubectl taint node calico-cluster-control-plane kubernetes.io/hostname=calico-cluster-control-plane:NoSchedule
```

## Testing a basic policy

```bash
kubectl create ns policy-demo
kubectl create deployment --namespace=policy-demo nginx --image=nginx
kubectl expose --namespace=policy-demo deployment nginx --port=80

```

We will use a busybox image to connect to the nginx pod.

```bash
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh

# From within the busybox container 
wget -q nginx -O -
```

Now we can enable isolation using a basic network policy.

```bash
kubectl apply -f calico/network-policy.yaml
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh

# From within the busybox container 
wget -q --timeout=5 nginx -O -
```
You shouldn't be able to access nginx.
Now let's allow the communication between nginx and the access container running a busybox.

```bash
kubectl apply -f calico/network-policy-access.yaml
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh

# From within the busybox container 
wget -q --timeout=5 nginx -O -
```

You should be getting acces to nginx's html page.
If we use another busybox container not matching label `access` then you shouldn't have access.
```bash
kubectl run --namespace=policy-demo noaccess --rm -ti --image busybox /bin/sh

# From within the busybox container 
wget -q --timeout=5 nginx -O -
# RESULT
#wget: download timed out
```

## Cleaning up resources

```bash
kubectl delete ns policy-demo
```



## Links
https://github.com/projectcalico/calico/releases
https://docs.projectcalico.org/reference/installation/api#operator.tigera.io/v1.Provider
https://docs.projectcalico.org/security/tutorials/kubernetes-policy-basic


