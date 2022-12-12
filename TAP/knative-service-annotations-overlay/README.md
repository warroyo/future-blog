# Knative Service Annotations

There are a few use cases where you may need to add annotations to a knative service. Currently the TAP OOTB supply chain does not allow for passing these in. This overlay will allow for an extra parameter to be passed at the workload level that can provide these annotations. This was mainly written to enable this annotation for [contour external auth](https://projectcontour.io/guides/external-authorization/) to work.

## Pre-reqs

* TAP cluster with TLS enabled. In order for this to work the workloads must use TLS



## Adding the overlay to TAP

To add this to a TAP install we will need to add some config the the TAP values file. Specifically we will be adding this to the `ootb-templates` package. The official docs for how to customize packages is [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.3/tap/GUID-customize-package-installation.html).



The first step is to create a secret with the overlay


```yaml
cat <<'EOF'  | kubectl  apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: knative-annotations-overlay
  namespace: tap-install
stringData:
  custom-package-overlay.yml: |
    #@ load("@ytt:overlay", "overlay")


    #@ def source_appconfig():
    kind: ClusterConfigTemplate
    metadata:
      name: config-template
    #@ end

    #@overlay/match by=overlay.subset(source_appconfig())
    ---
    spec:
      #@overlay/replace via=lambda left, right: left + "\n" + right
      ytt: |-
        #@ if hasattr(data.values.params,"ksvc_annotations"):

        #@ load("@ytt:overlay", "overlay")
        #@ load("@ytt:struct", "struct")
        #@ load("@ytt:yaml", "yaml")

        #@ def merge_annotations(existing,_):
        #@   ann = {}
        #@   current = dict(existing)
        #@   current.update(data.values.params["ksvc_annotations"])
        #@   return current
        #@ end

        #@ def shared_conf():
        metadata:
          #@overlay/replace via=merge_annotations
          annotations:
        #@ end

        #@ def replace_config(old, _):
        #@   return yaml.encode(overlay.apply(yaml.decode(old),shared_conf()))
        #@ end

        #@ def source_pipelinerun():
          kind: ConfigMap
        #@ end

        #@overlay/match by=overlay.subset(source_pipelinerun())
        ---
        data:
          #@overlay/replace via=replace_config
          delivery.yml:
        #@ end
EOF
```

We then need to update our tap values to reference the overlay secret as a package overlay on the `ootb-templates`. Update the tap values with the below and then re-apply it.


```yaml
package_overlays:
- name: ootb-templates
  secrets:
  - name: knative-annotations-overlay
```

```bash
 tanzu package installed update  tap -p tap.tanzu.vmware.com -v 1.3.2 --values-file tap-values.yaml -n tap-install
```


At this point the overlay should be applied and you can check to make sure the config-template has the updated code.

```bash
kubectl get clusterconfigtemplates config-template -o yaml
```

## Using the overlay to apply annotations to the service

We can now reference the new parameter in the workload yaml. Any annotations added to this parameter will be passed through to the `ksvc` object.

Here is an example workload yaml:

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  labels:
    app.kubernetes.io/part-of: company-api
    apps.tanzu.vmware.com/has-tests: "true"
    apps.tanzu.vmware.com/workload-type: web
  name: company-api
  namespace: default
spec:
  source:
    git:
      ref:
        branch: main
      url: https://github.com/warroyo/tap-go-sample
  params:
    - name: ksvc_annotations
      value:
        contour.networking.knative.dev/extension-service: "testing-service"
```


## Using this overlay to get contour external auth enabled

This is a use case that drove the creation of this overlay. In the next few steps we will deploy an external auth service for contour and then use the annotations to be able to specify the external auth service to be used with the knative service.


### Deploy the contour external auth service

We are just going to use the default example service. This can be used with any custom service though. [This Guide](https://projectcontour.io/guides/external-authorization/) outlines the process in detail. Follow these steps to get the service running. 


### Create a workload and reference the service


All we need to do is create the workload with the param set.

```bash
cat <<'EOF'  | kubectl  apply -f -
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  labels:
    app.kubernetes.io/part-of: company-api-htpass
    apps.tanzu.vmware.com/workload-type: web
  name: company-api-htpass
  namespace: default
spec:
  source:
    git:
      ref:
        branch: main
      url: https://github.com/warroyo/tap-go-sample
  params:
    - name: ksvc_annotations
      value:
        contour.networking.knative.dev/extension-service: "htpasswd"
        contour.networking.knative.dev/extension-service-namespace: "projectcontour-auth"
EOF
```

after this you can go to your workload url and should be prompted for a password.