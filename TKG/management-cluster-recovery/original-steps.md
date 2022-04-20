# MGMT cluster recovery


# steps

1. we need to be able to talk to the api first
    1. ssh to CP node and modify admin.conf 
        1. change the endpoint to https://localhost:6443
        2. add the `insecure-skip-tls-verify: true`
        3. comment out the cert
2. we noticed all SG rules were missing, rather than adding individual rules we added some broad rules to the SGs to allow for all traffic in the VPC and outbound.
    1. the hope is that once capa can reconcile it fixes the SG rules
3. all nodes are showing as not ready and we can’t pull logs for capa due to some errors.
    1. ssh to node with capa running and check logs on the filesystem
        1. we see errors about connection refused to coredns
4. now we need to get coredns healthy so we need to get atleast 1 node in a ready state
    1. ssh to capa worker and edit the kubelet.conf
        1. change the endpoint to https://localhost:6443
        2. add the `insecure-skip-tls-verify: true`
        3. comment out the cert
        4. restart kubelet `systemctl restart kubelet`
    2. node should show as ready now and. and now it should show endpoints behind the kube-dns service
    3. at this point capa is not throwing errors and starts to reconcile things
5. the duplicate cluster that was created is still lingering due to issues deleting in use objects which makes sense becuase the mgmt cluster is using it.
    1. delete the finalizers on the `awscluster`
    2. duplicate cluster goes away along with capa errors
6. after a few minutes CAPA created us a new LB however due to aws ELB DNS it has a new name so the api server is still down. we need to update everything to use the new name.
    1. backup and then delete both the capi and capa validatingwebhookconfigurations so we can update the `spec.controlplaneendpoint`  this allows us to modify the endpoint since it’s immutable
    2. you will also need to delete the capi mutating webhook
    3. edit the `awscluster` and `cluster` objects to use the new endpoint dns
    4. capi controller is still trying to use the old name
        1. edit the `<clustername>-kubeconfig` secret and change the endpoint as well as setting tls verify to off. 
        2. restart capi controller
    5. at this point capi is not throwing errors and starts reconciling all machines. machine creation will fail because the cert is still bad on the api server
    6. we also need to edit the following config maps to udpate the control plane endpoint
        1.  `kubectl edit cm -n kube-system kubeadm-config`
        2. `kubectl edit cm -n kube-system kube-proxy`
        3. `kubectl edit cm -n kube-public cluster-info`
7. now we need to fix certs 
    1. ssh to each CP node and run the following 
    2. `rm /etc/kubernetes/pki/apiserver.crt
    rm /etc/kubernetes/pki/apiserver.key
    kubeadm init phase certs apiserver --control-plane-endpoint="mytest.domain" --service-cidr=100.64.0.0/13 --cert-dir `
    1. update the CP endpoint to the new name in `/etc/kubernetes/kubelet.conf`
8. at this point capi should be able to create new worker nodes and CP nodes should be showing as healthy in the cluster
9. re-apply the deleted validating and mutating webhooks
10. restart antrea and kube-proxy
11. run the following to roll all nodes to make sure all configs are fresh
    1. `kubectl patch kcp <clusternamekcp> -n namespace --type merge -p "{\"spec\":{\"rolloutAfter\":\"`date +'%Y-%m-%dT%TZ'`\"}}"`
    2. `kubectl patch machinedeployment CLUSTER_NAME-md-0 --type merge -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"`