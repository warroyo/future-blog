# Recovering a Management Cluster in AWS


# Situation

With CAPA today it is possible to create a cluster in another namespace with the same name as one that already exist. This is problematic due to the way AWS resources are removed when deleting a cluster. The cleanup happens by finding the objects tagged with the owner which is done by cluster name, in this scenario the cluster name is now not a unique identifier and it will destroy AWS resources that are tied to another cluster. This can be extra problematic when you create a cluster with the same name as the management cluster. Here are the steps to recover from this situation where the mgmt cluster has become unstable due to the api server LB being destroyed.



# Steps

### **Access the api server locally**

1. ssh to a control plane node and modify the admin.conf

    ```
    vim /etc/kubernetes/admin.conf
    ```

2. replace the `server` with `server: https://localhost:6443`

3. add `insecure-skip-tls-verify: true`

4. comment out `certificate-authority-data:`

5. export the kubeconfig and ensure you can connect

    ```bash
    export KUBECONFIG=/etc/kubernetes/admin.conf`
    kubectl get nodes
    ```


### **Get rid of the lingerng duplicate cluster**

1. since there is a duplicate cluster that is trying to be deleted and can't due to some resources being unable to cleanup since they are in use we need to stop the conclficting reconciliation process. Edit the duplicate aws cluster object and remove the `finalizers`

    ```bash
    kubectl edit awscluster <clustername>
    ```
2. `kubectl get clusters` to verify it's gone


### **Make at least one node `Ready`**

1. right now all endpoints are down due to nodes not being ready. this is problematic for coredns adn antrea in particular. let's get one control plane node back healthy. on the control plane node we logged into edit the `kubelet.conf`

    ```bash
    vim /etc/kubernetes/kubelet.conf
    ```
2. replace the `server` with `server: https://localhost:6443`

3. add `insecure-skip-tls-verify: true`

4. comment out `certificate-authority-data:`

5. restart the kubelet `systemctl restart kubelet`

6. `kubectl get nodes` and validate that the node is in a  ready state.
7. After a few minutes most things should start scheduling themselves on the new node. The pods that did not restart on their own that were causing issues were core-dns,kube-proxy, and antrea.Those should be restart manually.
8. (optional) tail the capa logs to see the load balancer start to reconcile

    ```bash
    kubectl logs -f -n capa-system deployments.apps/capa-controller-manager`
    ```

### **Update the control plane nodes with new LB settings**

1. To be safe we will do this on all CP nodes rather than having them recreate to avoid potential data loss issues. Follow the following steps for **each** CP node.

2. Regenrate the certs for the api server using the new name. Make sure to update your service cidr and endpoint in the below command.

    ```bash
    rm /etc/kubernetes/pki/apiserver.crt
    rm /etc/kubernetes/pki/apiserver.key

    kubeadm init phase certs apiserver --control-plane-endpoint="mynewendpoint.com" --service-cidr=100.64.0.0/13 -v10
    ```

3. Update settings in `/etc/kubernetes/admin.conf`

    * Replace the `server` with `server: https://<your-new-lb.com>:6443`
    
    * Remove `insecure-skip-tls-verify: true`

    * Uncomment `certificate-authority-data:`

    * Export the kubeconfig and ensure you can connect 

        ```bash
        export KUBECONFIG=/etc/kubernetes/admin.conf
        kubectl get nodes
        ```

4. Update the settings in `/etc/kubernetes/kubelet.conf`

    * Replace the `server` with `server: https://your-new-lb.com:6443`

    * Remove `insecure-skip-tls-verify: true`

    * Uncomment `certificate-authority-data:`

    * restart the kubelet `systemctl restart kubelet`

5. Just as we did before we need new pods to pick up api server cache changes so  you will want to force restart pods like antrea, kube-proxy, core-dns , etc.

## 5. Update capi settings for new LB DNS name

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


## 6. Roll all of the nodes to make sure everything is fresh

1. `kubectl patch kcp <clusternamekcp> -n namespace --type merge -p "{\"spec\":{\"rolloutAfter\":\"`date +'%Y-%m-%dT%TZ'`\"}}"`
   
2. `kubectl patch machinedeployment CLUSTER_NAME-md-0 -n namespace --type merge -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"`