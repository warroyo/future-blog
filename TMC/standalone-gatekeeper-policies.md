# Applying Gatekeeper Policies Outside of TMC

When using TMC and creating policies TMC will automatically deploy gateekeeper into your k8s clusters. Today most of the underlying gatekeeper config is not user configurable which leads to some issues when trying to deploy a gatekeeper constraint outside of TMC's control. Specifically the gatekeeper webhook is dynamically updated by TMC to match specific resources based on the policies TMC is managing. This becomes a problem when deploying a constaint that wants to match against resources that TMC is not matching against.This will walk through how to get those policies to work by modifying the webhook to match against all resource types. If you start to see issues with scale by matching on `*` you can limit the webhook to only match on the resources you need. 

**NOTE: If you are using this in combination with TMC policies and not using `*` for the resources be sure to keep the webhook updated with the right resources. sepcifcally if using TMC security policy you will always want to include `pods`**

**WARNING: This workaround will cause the gatekeeper webhook to no longer be managed by TMC. the biggets risk here is that the CA cert gets rotated on gatekeeper due to a TMC upgrade and the webhook will start to fail. if this happens you will need to update the webhook with the new CA. This workaround is also not officially supported.**

with that out of the way here is how to make this work.

## Workaround Steps

First we need to get the current gatekeeper webhook config. the one we want is the `gatekeeper-validating-webhook-configuration`

```bash
kubectl get validatingwebhookconfigurations
NAME                                               WEBHOOKS   AGE
cert-manager-webhook                               1          189d
gatekeeper-validating-webhook-configuration        2          127d
istiod-istio-system                                1          233d
```

Now we need to patch the webhook so that it is no longer lifecycled by TMC.

```bash
kubectl annotate validatingwebhookconfigurations gatekeeper-validating-webhook-configuration tmc.cloud.vmware.com/managed-
```

Once this is done you can modify the webhook to match whichever resources you would like or `*`. the webhook config has 2 webhooks within it, the one we want to modify is `admissionReviewVersions` here is an example snippet of matching on `*`.

```bash
kubectl edit validatingwebhookconfigurations gatekeeper-validating-webhook-configuration
```

Then you can modify this section, notice the resources section. This is where we can match all or we can narrow it down.  

```yaml
rules:
  - apiGroups:
    - ""
    apiVersions:
    - '*'
    operations:
    - CREATE
    - UPDATE
    resources:
    - '*'
```