# Creating a custom workload type

This is going walk through creating a custom workload type in TAP. The use case we will use in this is having the ability to add volumes to a standard k8s deployment style workload. This is basically the `server` type workload with modifications. The way this is done is by creating a new `ClusterConfigTemplate` and registering that through the TAP values file as new supported  workload type. 


## Create a new config template

The config template will generate the deployment and service yaml. In this case we have taken the `server-template` and modified it to add the overlay defined in the `addVolumes` function that will inject volumes from the workload params. 

```bash
kubectl  apply -f volume-template.yml 
```


## Update the TAP values file

We need to update our tap values to add the new workload type. Add the config below. make sure to specify all of the type otherwise they will be removed.

```yaml
ootb_supply_chain_basic:
  ...
  supported_workloads:
  - type: web
    cluster_config_template_name: config-template
  - type: server
     cluster_config_template_name: server-template
  - type: worker
    cluster_config_template_name: worker-template
  - type: server-with-volumes
    cluster_config_template_name: volumes-template
```

```bash
 tanzu package installed update  tap -p tap.tanzu.vmware.com -v 1.3.2 --values-file tap-values.yaml -n tap-install
```


## Configure a workload to use the new type

We can now update the workload to use the new type with some volumes. The volumes params below each follow the upstream k8s volume spec.

```yaml
kind: Workload
metadata:
  creationTimestamp: "2022-12-01T00:08:43Z"
  generation: 4
  labels:
    app.kubernetes.io/part-of: go-sample
    apps.tanzu.vmware.com/has-tests: "true"
    apps.tanzu.vmware.com/workload-type: server-with-volumes
  name: go-sample2
  namespace: default

spec:
  params:
    - name: volumes
      value:
        volumes:
        - name: config-vol
          configMap:
            name: log-config
            items:
              - key: log_level
                path: log_level
        volumeMounts:
         - name: config-vol
           mountPath: /etc/config 
  source:
    git:
      ref:
        branch: master
      url: https://github.com/pacphi/go-gin-web-server
```
