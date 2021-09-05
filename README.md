# Gardener on Equinix Metal

This is a technical guide to running [Gardener](https://gardener.cloud) on [Equinix Metal](https://metal.equinix.com).

## What is Gardener

[Gardener](https://gardener.cloud) is a Kubernetes multi-cluster, multi-cloud, multi-region orchestration system. it allows
you to deploy and manage large numbers of Kubernetes clusters in many regions across the world, covering your own managed
on-premise clusters and multiple clouds, all from a single control pane.

This guide assumes familiarity with Gardener, and assists in setting up and running Gardener-managed clusters on Equinix Metal.

## Gardener Cluster Types

Gardener has three tiers of clusters, each of which serves a different purpose.

* Garden cluster
* Seed cluster
* Shoot cluster

We explain them in the reverse order.

### Shoot cluster

A shoot cluster is the cluster where you deploy your normal workloads: web servers, databases, machine learning, whatever workloads
you are trying to deploy. You will have many shoot clusters deployed all over the world across many providers and locations.

### Seed cluster

A seed cluster is responsible for managing one or more shoot clusters. It is a "middle tier" management cluster. Because of the
latency issues between cloud providers, and between regions within a single cloud provider, you generally have one seed cluster
per cloud provider per region. For example, if you have 15 shoot clusters, five deployed in each of three Equinix Metal metros
- ny, da, fr - then you would deploy one seed cluster in each facility to manage its local shoot clusters. If you also have
3 shoot clusters deployed in AWS us-east-1 and 4 deployed in your data centre in Sydney, Australia, then you would deploy
one additional seed cluster in each of those locations.

Seed clusters come in one of two forms:

* Seed - the Kubernetes cluster already is deployed, externally to Gardener, and the "seed" is deployed to the target cluster to turn it into a seed cluster
* Shooted Seed - Gardener deploys the actual Kubernetes cluster, and then the seed functionality to it

### Garden cluster

The garden cluster is the single, top-level management cluster. It is responsible for:

* managing the seed clusters and, through them, the shoot clusters
* interacting with end-users, allowing them to deploy seeds and shoots

## What Equinix Metal Supports

Equinix Metal supports the following:

* Shoots on Equinix Metal
* Seeds on Equinix Metal
* Shooted Seeds on Equinix Metal

Garden cluster on Equinix Metal is not yet supported. It is on the roadmap, but if this is a priority for you, please contact your account executive.

You can use a Garden cluster anywhere that Gardener is supported, and, from there, deploy seeds and/or shoots onto Equinix Metal. We have tested and approved
Garden cluster on several public clouds, and have written simplified guides. These guides are not materially different than the official Gardener docs,
but are simplified and will help you get started. The following are the supported clouds and guides.

* [Garden cluster on AWS/EKS](./aws)
* [Garden cluster on GCP/GKE](/gcp)

Once you have deployed a garden cluster - via one of the above guides or on your own - you should deploy a seed and a shoot.
There are several ways to get a Seed cluster:

* Depending on how you deployed your garden cluster, you might already have a seed deployed, as part of [garden-setup](https://github.com/gardener/garden-setup/)
* You can deploy a cluster outside of Gardener, and then convert it into a Seed, see [the official guide](https://gardener.cloud/docs/guides/install_gardener/setup-seed/)
* You can deploy a "shooted seed", i.e. Gardener will deploy a managed Shoot cluster directly from the garden cluster, and then convert it into a Seed, see [the official guide](https://github.com/gardener/gardener/blob/master/docs/usage/shooted_seed.md)

Deploying a Seed is beyond the scope of this document, please see the official guides referenced above.

Finally, you are ready to deploy Shoot clusters on Equinix Metal.

## Deploying an Equinix Metal Shoot Cluster

The steps are as follows:

1. Create and deploy a [Project](https://github.com/gardener/gardener/blob/master/docs/usage/projects.md)
1. From the Equinix Metal console or API, get your Project UUID and an API key
1. Create and deploy a `Secret` and a `SecretBinding` including the Project UUID and API key
1. Create and deploy a `CloudProfile`
1. Create and deploy a `Shoot`

We go through each step in turn.

### Create and Deploy a Project

A `Project` groups together shoots and infrastructure secrets in a namespace.
A sample `Project` is available at
[23-project.yaml](https://github.com/gardener/gardener-extension-provider-equinix-metal/tree/master/example/23-project.yaml).
Copy it over to a temporary workspace, modify it as needed, and then apply it.

```sh
kubectl apply -f 23-project.yaml
```

Unless you actually will be using the Gardener UI, most of the RBAC entries in the file do not matter for development.
The only really important elements are:

* `name`: pick a unique one for the `Project`
* `namespace`: you will need to be consistent in using the same namespace for multiple elements

### Get Your Project UUID and API Key

Each project in Equinix Metal has a unique UUID. You need to get that UUID in order to tell Gardener into which
Equinix Metal project it should deploy nodes.

When on the [Equinix Metal Console](https://console.equinix.com), and you select your project, you can see the UUID in the address bar,
for example, `https://console.equinix.com/projects/2331a81e-39f8-4a0f-8f82-2530d33e9b91` has the project UUID
`2331a81e-39f8-4a0f-8f82-2530d33e9b91`.

In addition, you need your API key. You can create new API keys, or find your existing ones, by clicking on your name in the upper-right
corner of the console, and then selecting "Personal API Keys" from the drop-down menu.

### Create and Deploy a `Secret` and a `SecretBinding`

In order to give Gardener access to your API key and Project UUID, you save them to a `Secret` and deploy them to the Seed cluster.
You also need a `SecretBinding`, which enables `Shoot` clusters to connect to the `Secret`.

A sample `Secret` and `SecretBinding` are available in
[25-secret.yaml](https://github.com/gardener/gardener-extension-provider-equinix-metal/tree/master/example/25-secret.yaml).
Copy it over to a temporary workspace, modify the following:

* `apiToken`  - the base64-encoded value of your Equinix Metal API key
* `projectID` - the base64-encoded value of your Equinix Metal Project UUID
* `namespace` - the namespace you provided in the Gardener Project in the previous step; this must be set for the `Secret` and the `SecretBinding`
* `name`      - the name of the `Secret` should be unique in the `namespace`, and the `secretRef.name` should match it in the `SecretBinding`

Then apply them:

```sh
kubectl -f 25-secret.yaml
```

### Create and Deploy a `CloudProfile`

The `CloudProfile` is a resource that contains the list of acceptable machine types, OS images, regions, and other information. When you deploy
actualy Shoot resources, they will match up to a `CloudProfile`.

A sample `CloudProfile` is available at
[26-cloudprofile.yaml](https://github.com/gardener/gardener-extension-provider-equinix-metal/tree/master/example/26-cloudprofile.yaml).
Copy it over to a temporary working directory, modify the following:

* `name` - a unique name for this cloud profile.
* `kubernetes.versions` - versions that will be supported.
* `machineImages` - OS images that will be supported. The `name` and `version` must match supported ones from Gardener, and must be in the `providerConfig`, further down.
* `machineTypes` - types of hardware servers that will be supported. The `name` must match the reference name from Equinix Metal.
* `regions` - supported Equinix Metal [metros](https://metal.equinix.com/developers/docs/locations/metros/). You also can have a list of `zones` in each `metro`, which must match Equinix Metal [facilities](https://metal.equinix.com/developers/docs/locations/facilities/) in that metro, but they are not required.
* `providerConfig.machineImages` - this is the list that maps Gardener-supported OS names and versions to Equinix Metal [Operating Systems](https://metal.equinix.com/developers/docs/operating-systems/supported/). The `name` and `version` must be supported by Gardener, and the `id` must be the Operating System ID supported by Equinix Metal.

Then apply it:

```sh
kubectl apply -f 26-cloudprofile.yaml
```

### Deploy Shoots

Finally, you are ready to deploy as many Shoot clusters as you want.

A sample Shoot cluster definition is available at [90-shoot.yaml](https://github.com/gardener/gardener-extension-provider-equinix-metal/tree/master/example/90-shoot.yaml).
Copy it over to a temporary working directory, and modify the following:

* `namespace` - must match the namespace of the Gardener project you deployed earlier
* `name` - must be a unique name for this Shoot
* `seedName` - this is optional, however, if your Seed is deployed in a different provider than your Shoot, e.g. your Seed is in GCP and your Shoot will be on Equinix Metal, then you must specify the `seedName` explicitly
* `secretBindingName` - must match the name of the `SecretBinding` you deployed earlier
* `cloudProfileName` - must match the name of the `CloudProfile` you deployed earlier
* `region` - must be one of the regions in the referenced `CloudProfile`
* `workers` - a list of worker pools. The `machine.type` and `image` must match those available in the referenced `CloudProfile`
* `kubernetes.version` - version of Kubernetes to deploy, which must match one of the versions in the referenced `CloudProfile`
* `networking` - adjust to your desired type and CIDR ranges

Then deploy it, make a nice drink, and await:

```sh
kubectl apply -f 90-shoot.yaml
```

