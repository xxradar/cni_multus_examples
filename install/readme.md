## Setting up multus 

### Create a K8S cluster
After you create a K8S cluster, verify everything is up and running.

### Installing Multus
This is purely based on  https://github.com/intel/multus-cni/blob/master/doc/quickstart.md
```
git clone https://github.com/intel/multus-cni.git && cd multus-cni
```
Multus will be installed as daemonset. Pleqse note that there probably already an installed CNI.
```
kubectl apply -f ./multus-cni/images/multus-daemonset.yml
```
You can validate the istall 
```
kubectl get pods --all-namespaces | grep -i multus
```
### Creating addition CNI configurations
The idea behind Multus is to mount multiple interfaces in a pod. There for you must have something in place that can connect an interface to a network (ex. bridge or interface) and manage the IP addresses (preferably accross the cluster). This is  configured in a additional CNI configuration.

### Network attachment via macvlan cni plugin
```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
EOF
```
```
kubectl get network-attachment-definitions
```
```
kubectl describe network-attachment-definitions macvlan-conf
```
#### Let's create a sample pod to verify
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hackon1
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: hackon
    command: ["/bin/bash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: xxradar/hackon
EOF
```
If you sping up multiple pods (assuming your nodes are in the same L2 network or node) they will have connectivity accross the newly mounted net1 interface.

### Network attachment via bridge cni plugin
```
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-conf
spec:
  config: '{
    "cniVersion": "0.3.0",
    "name": "bridge-conf",
    "type": "bridge",
    "bridge": "br0",
    "isGateway": true,
    "ipam": {
     "type": "host-local",
     "subnet": "192.168.99.0/24",
     "dataDir": "/mnt/cluster-ipam"
    }
}'
EOF
```
#### Let's create a sample pod to verify
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hackon4
  annotations:
    k8s.v1.cni.cncf.io/networks: bridge-conf
spec:
  containers:
  - name: hackon
    command: ["/bin/bash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: xxradar/hackon
EOF
```
You can verify the interfaces and IP addresses in the pods. <br>
Also note that on the node the pod is created, a bridge called br0 is created.
```
brctl show
```
#### Let's create a sample pod a mount multiple interfaces
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hackon8
  annotations:
     k8s.v1.cni.cncf.io/networks: bridge-conf, macvlan-conf
spec:
  containers:
  - name: hackon
    command: ["/bin/bash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: xxradar/hackon
EOF
```

