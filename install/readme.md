## Setting up multus 

### Create a K8S cluster
After you create a K8S cluster, verify everything is up and running


### Installing Multus
This is purely based on  https://github.com/intel/multus-cni/blob/master/doc/quickstart.md
```
git clone https://github.com/intel/multus-cni.git && cd multus-cni
```
Multus will be installed as daemonset. Pleqse note that there probably already an installed CNI.
```
cat ./images/multus-daemonset.yml | kubectl apply -f -
```
You can validate the istall 
```
kubectl get pods --all-namespaces | grep -i multus
```
