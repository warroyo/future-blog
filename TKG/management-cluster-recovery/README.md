# Recovering a Management Cluster in AWS


# Situation

With CAPA today it is possible to create a cluster in another namespace with the same name as one that already exist. This is problematic due to the way AWS resources are removed when deleting a cluster. The cleanup happens by finding the objects tagged with the owner which is done by cluster name, in this scenario the cluster name is now not a unique identifier and it will destroy AWS resources that are tied to another cluster. This can be extra problematic when you create a cluster with the same name as the management cluster. Here are the steps to recover from this situation where the mgmt cluster has become unstable due to the api server LB being destroyed.



# Steps


## 1. Access the api server locally

ssh to a control plane node and modify the admin.conf

```
vim /etc/kubernetes/admin.conf
```

replace the `server` with `server: https://localhost:6443`

add `insecure-skip-tls-verify: true`

comment out `certificate-authority-data:`

export the kubeconfig and ensure you can connect `kubectl get nodes`

`export KUBECONFIG=/etc/kubernetes/admin.conf`

 `kubectl get nodes`



## Get rid of the lingerng duplicate cluster

since there is a duplicate cluster that is trying to be deleted and can't due to some resources being unable to cleanup since they are in use we need to stop the conclficting reconciliation process. Edit the duplicate aws cluster object and remove the `finalizers`

`kubectl edit awscluster <clustername>`

it should be deleted shirtly after

`kubectl get clusters` to verify it's gone


## Make at least one node `Ready`

right now all endpoints are down due to nodes not being ready. this is problematic for coredns adn antrea in particular. let's get one control plane node back healthy.

on the control plane node we logged into edit the `kubelet.conf`

`vim /etc/kubernetes/kubelet.conf`

replace the `server` with `server: https://localhost:6443`

add `insecure-skip-tls-verify: true`

comment out `certificate-authority-data:`


restart the kubelet

`systemctl restart kubelet`


you should have a node in the `Ready` state now.


After a few minutes most things should start scheduling themselves on the new node.The pods that did not restart on their own that were causing issues were core-dns,kube-proxy, and antrea. Those should be restart manually.

tail the capa logs to see the load balancer start to reconcile

`kubectl logs -f -n capa-system deployments.apps/capa-controller-manager`


## Update the control plane nodes with new LB settings

To be safe we will do this on all CP nodes rather than having them recreate. follow these steps for each CP node.

make sure to update your service cidr and endpoint in the below command.Use this command to regenrate the certs for the api server using the new name. 

```
rm /etc/kubernetes/pki/apiserver.crt
rm /etc/kubernetes/pki/apiserver.key

kubeadm init phase certs apiserver --control-plane-endpoint="rvzpn1iy11dda8cy7gyex082zicw-k8s-1780442051.us-west-2.elb.amazonaws.com" --service-cidr=100.64.0.0/13 -v10
```

update setiings admin.conf and kubelet.conf

```
vim /etc/kubernetes/admin.conf
```

replace the `server` with `server: https://your-new-lb.com:6443`

remove `insecure-skip-tls-verify: true`

uncomment `certificate-authority-data:`

export the kubeconfig and ensure you can connect `kubectl get nodes`

`export KUBECONFIG=/etc/kubernetes/admin.conf`

 `kubectl get nodes`



`vim /etc/kubernetes/kubelet.conf`

replace the `server` with `server: https://your-new-lb.com:6443`

remove `insecure-skip-tls-verify: true`

uncomment `certificate-authority-data:`


restart the kubelet

`systemctl restart kubelet`

just as we did before we need new pods to pick up api server cache changes so  you will want to force restart pods like antrea, kube-proxy, core-dns , etc.

## Update capi settings for new LB DNS name

Update the control plane endpoint on the `awscluster` and `cluster` objects. To do this we need to disable the validatingwebhooks.we will back them up and then delete so we can apply later.

```
kubectl get validatingwebhookconfigurations capa-validating-webhook-configuration -o yaml > capa-webhook && kubectl delete validatingwebhookconfigurations capa-validating-webhook-configuration

kubectl get validatingwebhookconfigurations capi-validating-webhook-configuration -o yaml > capi-webhook && kubectl delete validatingwebhookconfigurations capi-validating-webhook-configuration
```

edit the `spec.controlPlaneEndpoint.host` field on both `awscluster` and `cluster` to have the new endpoint

re-apply your webhooks


Update the following config maps and replace the old control plane name with the new one.

```bash
kubectl edit cm -n kube-system kubeadm-config
kubectl edit cm -n kube-system kube-proxy
kubectl edit cm -n kube-public cluster-info
```

edit the cluster kubeconfig secret that capi uses to talk to the mgmt cluster. you will need to decode teh secret, replace the endpoint and re-encode and save.

`kubectl edit secret -n tkg-system <cluster-name>-kubeconfig`

at this point things should start to reconcile on thier own, but we can use the commands in the next step to force it. 


## Roll all of the nodes to make sure everything is fresh

1. `kubectl patch kcp <clusternamekcp> -n namespace --type merge -p "{\"spec\":{\"rolloutAfter\":\"`date +'%Y-%m-%dT%TZ'`\"}}"`
   
2. `kubectl patch machinedeployment CLUSTER_NAME-md-0 -n namespace --type merge -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"`