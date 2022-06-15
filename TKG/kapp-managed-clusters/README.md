# Declarative TKG Cluster Updates Using Kapp

While using TKG  you may want to update settings for a cluster after the initial creation. The Tanzu CLI does not have a way to do this easily today. The Tanzu CLI does have a `--dry-run` option that will output all of the yaml that gets applied to a cluster after running it through the necessary ytt overlays etc. The challenge with this however is that there are a number of immutable fields that Cluster Api has and also injects new immutable fields into the spec after the yaml is applied. This is problematic because when you generate the yaml and apply it to try and make an update some will resource updates will fail due to immutable fields changing. There has been some fantastic work done by the folks listed below around deploying clusters with GitOps and using the [kapp](https://carvel.dev/kapp/) that solves these problems. The reason for creating this doc is to show how to avoid the above mentioned issues and give the ability to update clusters easily using the Tanzu CLI for those who have not gone down the full GitOps route and would like to use a cli to push updates but still be able to do it in a declarative way. The real magic here is all in how Kapp works. This will even allow for updates to things like machinetemplates which are immutable by using the [kapp versioned resources feature](https://carvel.dev/kapp/docs/v0.45.0/diff/#versioned-resources). 


## Demo

this video shows vertically scaling the control plane machines and hwo kapp handles versioned resources to make this really easy.



https://user-images.githubusercontent.com/616621/163892454-e1ccc39e-4a14-4515-890b-cf6fc7b6c053.mp4



## Usage


### Prerequisites

* Kapp cli
* Tanzu cli
* A TKG management cluster


### Setup

Copy the following 2 files into your tkg providers directory. This sets up an overlay to add kapp specific config to the objects as well as configures Kapp's deployment behavior.

```bash
cp kapp-overlay.yml ~/.config/tanzu/tkg/providers/ytt/03_customizations/
cp kapp-overlay-values.yml ~/.config/tanzu/tkg/providers/ytt/03_customizations/
```

This will expose a new value for your cluster config `KAPP_MANAGED` setting this to true will turn on the new overlays. This allows you to only use this method for clusters you determine.


### Usage

1. add `KAPP_MANAGED: "true"` to your cluster config.
2. run the following command.

```
tanzu cluster create <cluster-name> -f <cluster-config-yaml> --dry-run | kapp deploy --diff-changes  -a <cluster-name> --wait=false -y  -f -
```

This will generate the cluster yaml and then send it to kapp which will handle the apply to the cluster using a customized set of rules to handle any issues with immutable objects etc. This same command can now be used for cluster creates as well as updates since this is declarative.


### Using with Management CLusters

You can manage Management clusters with kapp but it needs to be done after the initial creation. These steps assume you have already created a manangement cluster with `tanzu mc create...` and haave now swicthed into the kube context of that management cluster.

*** NOTE: if you created the management cluster without providing an existing VPC you will need to update your cluster config yaml file to have the automatically created VPC info(id, subnets) in the config otherwise it will complain about a missing VPC ID. ***

1. set your namespace in the cluster config, this will be `NAMESPACE: tkg-system` for management clusters
2. set the environment variable `export _TKG_CLUSTER_FORCE_ROLE="management"`  this forces the cli to create the yaml for a mgmt cluster
3. run the following command

```
tanzu cluster create <mgmt-cluster-name> -f <mgmt-cluster-config-yaml> --dry-run | kapp deploy --diff-changes  -a <mgmt-cluster-name> --wait=false -y  -f -

```

### Deleting Clusters

In order to clean up all of the resources properly you should use kapp for the delete as well.

`kapp list` -  this will show all clusters that are managed

`kapp delete -a <cluster-name>` - this will output all of the resources and prompt for deletion


### FAQ

1. if your cluster was initially created without kapp and are seeing some lingering `machineTemplates` and `KubeadmConfigTemplates` that show up as "skipped" in the kapp output this is due to the original resources needing to exist for the initial onboarding to kapp. Once your cluster has been kapp managed and the machineDeployments are using versioned `machineTemplates` you can clean these up if you don't want to see  them in the output. they are harmless though. 

## Special Thanks

Thanks to some of my coworkers who have done some great work using TKG and Kapp. This is possible due to their work and they did the heavy lifting on Kapp configuration used here. 

[Brian Durden](https://github.com/bcdurden)

[Conzetti Finocchiaro](https://github.com/conzetti)

[Robert Van Vorhees](https://github.com/voor)
