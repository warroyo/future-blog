# Tanzu App Catalog Rabbit MQ with External Secret Operator

This walks through using the TAC RabbitMQ helm chart with secrets provided by AKV(azure key vault) and external secrets operator. Specifically it will go over adding users to RabbitMQ that are defined in AKV.

## pre-reqs

* a functional AKV
* ESO installed in the cluster
* a secret store or cluster secret store connected to the AKV.

## Setup the secrets in AKV

```bash
az keyvault secret set --vault-name "dev-environ" --name "user1" --value "complicatedPassword"
az keyvault secret set --vault-name "dev-environ" --name "user2" --value "complicatedPassword2"
```
## Create the External Secret

To use values from an external secret we can utilize the [templating functionality](https://external-secrets.io/v0.9.13/guides/templating/) of ESO. This will let us pull in secrets from AKV and use them as variables when generating the resulting secret. In this case the resulting secret will be a rabbit load defintion as defined [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Catalog/services/apps/GUID-apps-rabbitmq-index.html#load-custom-definitions). However rather than using the helm chart and storing the secrets in plain text we will use ESO templating to create the secret and then reference it in the chart values as a pre-existing secret.

Apply the following external secret template defintion into your cluster/namespace. 

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rabbit-load-definition
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: cluster-wide-store
  target:
    name: rabbit-load-definition
    template:
      engineVersion: v2
      data:
        # multiline string
        load_definition.json: |
          {
            "users": [
            {
                "name": "user1",
                "password": "{{ .user1-password }}",
                "tags": "user1"
            },
            {
                "name": "user2",
                "password": "{{ .user2-password }}",
                "tags": "user2"
            }
            ]
          }
  data:
  - secretKey: user1-password
    remoteRef:
      key: user1
  - secretKey: user2-password
    remoteRef:
      key: user2

```

validate that the secret was created. you should see a sucessful secret sync on the `externalSecret`.

```bash
kubectl describe externalsecret rabbit-load-definition -n <namespace>
```

validate that the secret contents looks correct. you should see the load_definition.json contents after running the following command, with the secrets injected.

```bash
kubectl get secret rabbit-load-definition -n <namespace> -o json | jq -r '.data."load_definition.json"' | base64 -d

{
  "users": [
  {
      "name": "user1",
      "password": "complicatedPassword",
      "tags": "user1"
  },
  {
      "name": "user2",
      "password": "complicatedPassword2",
      "tags": "user2"
  }
  ]
}
```


## Create the values file

Now we need to  create the values file that will reference the secret we just created. This will not walk through all the values needed to deploy the chart, just the ones that need to be added to use this secret.

add the following to your values file:

```yaml
loadDefinition:
  enabled: true
  existingSecret: rabbit-load-definition
extraConfiguration: |
  load_definitions = /app/load_definition.json
```

Notice the reference to the secret above, as well as using the secret's key `load_definiton.json` as the file name in the `extraConfiguration`.


## Deploy the chart using the updated values file

Follow docs [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Catalog/services/apps/GUID-apps-rabbitmq-index.html#installing-the-chart) for install.

Once the chart is deployed check to see if the users are created.

you should see the users created in the logs, for example:

```
2024-02-27 00:33:23.242757+00:00 [info] <0.525.0> Importing concurrently 2 users...
2024-02-27 00:33:23.244968+00:00 [info] <0.516.0> Successfully changed password for user 'user1'
2024-02-27 00:33:23.244966+00:00 [info] <0.517.0> Successfully changed password for user 'user2'
2024-02-27 00:33:23.245047+00:00 [info] <0.517.0> Successfully set user tags for user 'user2' to [user2]
2024-02-27 00:33:23.245030+00:00 [info] <0.516.0> Successfully set user tags for user 'user1' to [user1]
2024-02-27 00:33:23.245201+00:00 [info] <0.525.0> Ready to start client connection listeners
2024-02-27 00:33:23.275198+00:00 [info] <0.698.0> started TCP listener on [::]:5672
```