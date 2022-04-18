# Declarative Cluster Updates Using Kapp

While using TKG  you may want to update settings for a cluster after the initial creation. The Tanzu CLI does not have a way to do this easily today. The Tanzu CLI does have a `--dry-run` option that will output all of the yaml that gets applied to a cluster after running it through the necessary ytt overlays etc. The challenge with this however is that there are a number of immutable fields that Cluster Api has and also injects new immutable fields into the spec after the yaml is applied. This is problematic because when you generate the yaml and apply it to try and make an update some will resource updates will fail due to immutable fields changing. There has been some fantastic work done by the folks listed below around deploying clusters with GitOps and using the [kapp](https://carvel.dev/kapp/) that solves these problems. The reason for creating this doc is to show how to avoid the above mentioned issues and give the ability to update clusters easily using the Tanzu CLI for those who have not gone down the full GitOps route and would like to use a cli to push updates but still be able to do it in a declarative way. The real magic here is all in how Kapp works. This will even allow for updates to things like machinetemplates which are immutable by using the [kapp versioned resources feature](https://carvel.dev/kapp/docs/v0.45.0/diff/#versioned-resources). 

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


### Deleting Clusters

In order to clean up all of the resources properly you should use kapp for the delete as well.

`kapp list` -  this will show all clusters that are managed

`kapp delete -a <cluster-name>` - this will output all of the resources and prompt for deletion

## Special Thanks

Thanks to some of my coworkers who have done some great work using TKG and Kapp. This is possible due to their work and they did the heavy lifting on Kapp configuration used here. 

[Brian Durden](https://github.com/bcdurden)

[Conzetti Finocchiaro](https://github.com/conzetti)

[Robert Van Vorhees](https://github.com/voor)