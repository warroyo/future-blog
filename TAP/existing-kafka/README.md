# Connecting a workload to an Kafka instance

This article cover creating the required resources to bind a workload to an existing kafka instance using the services toolkit. This kafka instance is using SCRAM auth and is an AWS MSK instance.


## Create a Binding Specification Compatible Secret

We need a secret that will contain our binding details for kafka

1. create the rds db secret

```bash
cat <<'EOF'  | kubectl -n dev apply -f - 
apiVersion: v1
kind: Secret
metadata:
  name: kafka-creds
  labels:
    app.kubernetes.io/component: existing-kafka
    app.kubernetes.io/instance: ""
    services.apps.tanzu.vmware.com/class: existing-kafka
type: Opaque
stringData:
  type: kafka
  BootstrapServers: b-3-public.accuriskafkademo.alvicg.c2.kafka.us-west-2.amazonaws.com:9196,b-1-public.accuriskafkademo.alvicg.c2.kafka.us-west-2.amazonaws.com:9196,b-2-public.accuriskafkademo.alvicg.c2.kafka.us-west-2.amazonaws.com:9196
  SaslMechanism: "ScramSha512"
  SecurityProtocol: "SaslSsl"
  SaslPassword: 'securekafka'
  SaslUsername: 'accuris'

EOF
```

## Create a Service instance class

We need to create this instance class to make the kafka instance discoverable for usage with clas claims. 
1. Create the cluster instance class. Notice the label selector matches our label on the secret  above.

```bash
cat <<'EOF'  | kubectl -n dev apply -f - 
# clusterinstanceclass.yaml
---
apiVersion: services.apps.tanzu.vmware.com/v1alpha1
kind: ClusterInstanceClass
metadata:
  name: existing-kafka
spec:
  description:
    short: AWS MSK kafka instance
  pool:
    kind: Secret
    labelSelector:
      matchLabels:
        services.apps.tanzu.vmware.com/class: existing-kafka
EOF
```

2. Create a role binding to allow the services toolkit to read the  secret

```bash
cat <<'EOF'  | kubectl -n default apply -f - 
# stk-secret-reader.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: stk-secret-reader
  labels:
    servicebinding.io/controller: "true"
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - list
  - watch
EOF
```


3. validate that we have a service class available now

```bash
tanzu services classes list
```


## Create a class claim

```bash
cat <<'EOF'  | kubectl -n dev apply -f - 

apiVersion: services.apps.tanzu.vmware.com/v1alpha1
kind: ClassClaim
metadata:
  name: kafka-1
spec:
  classRef:
    name: existing-kafka
  parameters: {}
EOF
```

check that it exists

```bash
tanzu service class-claim list -n dev
```

1. add the claim to your workload yaml

```yaml
serviceClaims:
- name: kafka
  ref:
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ClassClaim
    name: kafka-1
```