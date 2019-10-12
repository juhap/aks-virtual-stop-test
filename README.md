(This is related to AKS issue: https://github.com/Azure/AKS/issues/1267)

A quick test to see how the Azure AKS with virtual nodes handles the delay in preStop lifecycle hook. Based on this
there seems to be difference between the regular nodes and virtual nodes. In the virtual node the pod gets killed immediately, 
while the regular node behaves as expected.

First we create the pods. The deployed container is dummy helloworld webapp. 
```bash
$ kubectl apply -f hello_normal.yaml                                                                      
deployment.apps/hello-normal created

$ kubectl apply -f hello_virtual.yaml                                                                     
deployment.apps/hello-virtual created
```

Check pods are running, verify the other is running on the ACI virtual node.
```bash
$ kubectl get pods                                                                                         
NAME                             READY   STATUS     RESTARTS   AGE
hello-normal-85f5c44d87-vw2x4    1/1     Running    0          11s
hello-virtual-64d7d486c8-575r4   1/1     Running    0          5s

$ kubectl get pod -o=custom-columns=NODE:.spec.nodeName,NAME:.metadata.name --all-namespaces |grep hello
aks-nodepool1-33988429-vmss000000   hello-normal-85f5c44d87-vw2x4
virtual-node-aci-linux              hello-virtual-64d7d486c8-575r4
```

Delete the pod. The deletion is expected to take some time, as we have "sleep 45" in the preStop lifecycle hook for the container.
Even if the container itself will shutdown immediately, the preStop hook should block the container from being shutdown while it is 
executing.

```bash
$ time -p kubectl delete pod hello-normal-85f5c44d87-vw2x4                                           
pod "hello-normal-85f5c44d87-vw2x4" deleted
real 50.46
user 0.18
sys 0.09
```

```bash
$ time -p kubectl delete pod hello-virtual-64d7d486c8-575r4                                               
pod "hello-virtual-64d7d486c8-575r4" deleted
real 1.12
user 0.15
sys 0.12
```

As the execution times show, the container deployed in regular node takes ~50 seconds to stop. The other container deployment to 
ACI virtual node stops immediately.
