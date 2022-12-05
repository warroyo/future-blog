# Connecting a workload to an existing RDS MYSQL instance

This article covers how to use ACK and the TAP services toolkit to connect to an existing RDS instance in AWS. This is implemented on an EKS cluster so if you are using a non EKS cluster some of the steps for the ACK install will be different. 


## Installing ACK

We will not run through the entire install of ack here since it is documented nicely in the AWS docs. follow the Guide [here](https://aws-controllers-k8s.github.io/community/docs/user-docs/install/). There are only two changes to make.

1. the Service should be rds instead of s3
```
export SERVICE=rds
```

2. be sure to set your region accordingly

```
export AWS_REGION=us-west-2
```

Make sure to configure IAM permissions documented [here](https://aws-controllers-k8s.github.io/community/docs/user-docs/irsa/)


## Adopting an existing RDS instance with ACK

Ack provides a CRD for adopting existing resources. We will use this to create a DBInstance from an existing RDs instance.

Create the Adoption yaml and apply this into your cluster. Replace the `INSTANCE_NAME` with your database name . This will find the resource in the `aws` section and then stamp out the resource in the `kubernetes` section.

```yaml
apiVersion: services.k8s.aws/v1alpha1
kind: AdoptedResource
metadata:
  name: tap-db-adopt
spec:  
  aws:
    nameOrID: INSTANCE_NAME
  kubernetes:
    group: rds.services.k8s.aws
    kind: DBInstance
    metadata:
      name: tap-mysql-db
      namespace: default
```

If this works you will see a DBInstance when running `kubectl get dbinstance -n default`


## Create a Binding Specification Compatible Secret

Because we adopted an instance we do not have a secret that exists in the cluster for the DB password. We will need to create this and then create our secret template.

1. create the rds db secret

```bash
cat <<'EOF'  | kubectl -n default apply -f - 
apiVersion: v1
kind: Secret
metadata:
  name: tap-mysql-db-password
type: Opaque
stringData:
  password: 'PASSOWRD_HERE'
EOF
```

2. Create a service account for secret templating

```bash
cat <<'EOF'  | kubectl -n default apply -f - 
---
apiVersion: v1
kind: ServiceAccount
metadata:
 name: rds-resources-reader
 namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
 name: rds-resources-reading
 namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - list
  - watch
  resourceNames:
  - tap-mysql-db-password
- apiGroups:
  - rds.services.k8s.aws
  resources:
  - dbinstances
  verbs:
  - get
  - list
  - watch
  resourceNames:
  - tap-mysql-db
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
 name: rds-resources-reader-to-read
 namespace: default
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: Role
 name: rds-resources-reading
subjects:
 - kind: ServiceAccount
   name: rds-resources-reader
   namespace: default
EOF
```

3. Create a secret template, replace the `EXISTING DB HERE` with an existing db in your RDS instance.

```bash
cat <<'EOF'  | kubectl -n default apply -f - 
# bindable-rds-secrettemplate.yaml
---
apiVersion: secretgen.carvel.dev/v1alpha1
kind: SecretTemplate
metadata:
 name: rds-bindable
 namespace: default
spec:
 serviceAccountName: rds-resources-reader
 inputResources:
 - name: rds
   ref:
     apiVersion: rds.services.k8s.aws/v1alpha1
     kind: DBInstance
     name: tap-mysql-db
 - name: creds
   ref:
     apiVersion: v1
     kind: Secret
     name: "tap-mysql-db-password"
 template:
  metadata:
    labels:
      app.kubernetes.io/component: rds-mysql
      app.kubernetes.io/instance: "$(.rds.metadata.name)"
      services.apps.tanzu.vmware.com/class: rds-mysql
  type: mysql
  stringData:
    type: mysql
    port: "$(.rds.status.endpoint.port)"
    database: "EXISTING DB NAME"
    host: "$(.rds.status.endpoint.address)"
    username: "$(.rds.spec.masterUsername)"
  data:
    password: "$(.creds.data.password)"
EOF
```

Validate that the secret was created.
```
kubectl get secrettemplate -n default rds-bindable -o jsonpath="{.status.secret.name}"
```

## Create a Service instance class

We need to create this instance class to make the db instance discoverable for usage with resource claims. 
1. Create the cluster instance class. Notice the label selector matches our label on the secret template above.

```bash
cat <<'EOF'  | kubectl -n default apply -f - 
# clusterinstanceclass.yaml
---
apiVersion: services.apps.tanzu.vmware.com/v1alpha1
kind: ClusterInstanceClass
metadata:
  name: aws-rds-mysql
spec:
  description:
    short: AWS RDS instances with a mysql engine
  pool:
    kind: Secret
    labelSelector:
      matchLabels:
        services.apps.tanzu.vmware.com/class: rds-mysql
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

## Create a resource claim

Now that we have a service class available we need to create a resource claim and bind it to our workload.


1. Create a claim

```bash
tanzu service claim create mysql-rds-claim \
  --resource-name rds-bindable \
  --resource-kind Secret \
  --resource-api-version v1
```

validate it created

```bash
tanzu service claim list -o wide
```


2. add the claim to your workload yaml

```yaml
serviceClaims:
- name: db
  ref:
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ResourceClaim
    name: mysql-rds-claim
```