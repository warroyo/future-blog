# Quick TLS with Cert Manager

For TAP POCs and quick installs it may be useful to have a simple way to setup TLS for all ingress. This walks through setting upa  CA-issuer with a self signed cert in cert-manager and applying it to your TAP install.

## Create a self signed CA

This will create a self signed cluster issuer that will be the source for our generated CA.

```bash
cat <<'EOF'  | kubectl  apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
 name: selfsigned
spec:
 selfSigned: {}
EOF
```

We then use that self signed issuer to issue a self signed CA cert, that we can then use for our CA issuer. You could also use a CA generated outside of cert manager but this makes it quick and easy. Just replace `YOUR_TAP_DOMAIN` with your base tap domain.

```bash
cat <<'EOF'  | kubectl  -n cert-manager apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
 name: self-signed-ca-tap
spec:
 secretName: self-signed-tap-tls-ca-key-pair
 isCA: true
 issuerRef:
   name: selfsigned
   kind: ClusterIssuer
 commonName: "YOUR_TAP_DOMAIN"
 dnsNames:
 # one or more fully-qualified domain name
 # can be defined here
 - YOUR_TAP_DOMAIN
EOF
```


## Create a CA issuer based on our new self signed CA

Now we can reference the CA that was created as our base secret to generate new certs using a CA cluster issuer. 

```bash
cat <<'EOF'  | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
 name: ca-issuer-tap
spec:
 ca:
   secretName: self-signed-tap-tls-ca-key-pair
EOF
```


At this point you should validate that your `ClusterIssuer` is in a `ready` state.


## Applying the issuer to CNRS

### For TAP <= 1.3

In TAP `1.3.x` and below the we need to use an overlay to apply the issuer. Follow these steps.


1. Create an overlay that will update the issuer on the CNRs package

```bash
cat <<'EOF'  | kubectl  -n tap-install apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: cnrs-tls-overlay
  namespace: tap-install
stringData:
  custom-package-overlay.yml: |
    #@ load("@ytt:overlay", "overlay")


    #@overlay/match by=overlay.subset({"kind":"ConfigMap", "metadata":{"name":"config-certmanager","namespace":"knative-serving"}})
    ---
    apiVersion: v1
    kind: ConfigMap
    data:
    #@overlay/match missing_ok=True
    issuerRef: |
        kind: ClusterIssuer
        name: ca-issuer-tap
    #@overlay/remove missing_ok=True
    _example:

    #@overlay/match by=overlay.subset({"kind":"ConfigMap", "metadata":{"name":"config-network","namespace":"knative-serving"}})
    ---
    apiVersion: v1
    kind: ConfigMap
    data:
    #@overlay/match missing_ok=True
    autoTLS: "Enabled"
    #@overlay/match missing_ok=True
    httpProtocol: "Redirected"
    #@overlay/remove missing_ok=True
    _example:


    #@ def kapp_config():
    apiVersion: kapp.k14s.io/v1alpha1
    kind: Config
    #@ end

    #@overlay/match by=overlay.subset(kapp_config())
    ---
    rebaseRules:
    #@overlay/append
    - path: [data]
    type: copy
    sources: [new, existing]
    resourceMatchers:
    - kindNamespaceNameMatcher: {kind: ConfigMap, namespace: knative-serving, name: config-domain}
    - kindNamespaceNameMatcher: {kind: ConfigMap, namespace: knative-serving, name: config-network}
    - kindNamespaceNameMatcher: {kind: ConfigMap, namespace: knative-serving, name: config-certmanager}
    - kindNamespaceNameMatcher: {kind: ConfigMap, namespace: knative-serving, name: config-autoscaler}
    - kindNamespaceNameMatcher: {kind: ConfigMap, namespace: knative-serving, name: config-contour}
EOF
```

2. In your TAP values file add the following. 

```yaml
package_overlays:
- name: cnrs
  secrets:
  - name: cnrs-tls-overlay
```


Now when you deploy a workload it should get a cert automatically.