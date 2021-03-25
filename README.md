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

