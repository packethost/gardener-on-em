# Deploying a Garden cluster on GCP/GKE

This is based upon the [official garden-setup guide](https://github.com/gardener/garden-setup), as well as the
[Gardener on AWS guide](https://gist.github.com/johananl/8e2daec6c6f772fb03846425ca387aca).

Notes:

* it doesn't matter if we use route-based or IP-Alias networking for GKE clusters, but this guide uses IP-Alias

This is the process. Anything that requires additional details either has a link or is further below in the document.

1. Deploy a GKE cluster, using whatever method works for you: [Web console](https://console.cloud.google.com/kubernetes/), [CLI](https://cloud.google.com/sdk/docs/cheatsheet#docker_google_kubernetes_engine_gke), [Terraform](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_cluster), etc.
1. Deploy a nodepool of at least 4 nodes, with at least 8 GB for each node, and wait for it to deploy; you may eventually need more, like 6 or 8
2. if not already installed, install [gcloud CLI](https://cloud.google.com/sdk/gcloud), also available via [homebrew on macOS](https://formulae.brew.sh/cask/google-cloud-sdk)
3. get a [local kubeconfig for the GKE cluster](https://cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials)
4. deploy the [vertical pod autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) CRDs. 
5. Get the [sow](https://github.com/gardener/sow.git) repository. Yes, unfortunately, you need to clone the whole thing. `git clone https://github.com/gardener/sow.git && cd sow`
6. Add the `sow` command to your path: `export PATH=$PATH:$(pwd)/docker/bin`
7. Make a `landscape` subdirectory and clone [garden-setup](https://github.com/gardener/garden-setup.git) into a subdirectory named `crop` in it. Yes, we are cloning `garden-setup` into a subdirectory called `crop`, but that is what we need to do: `mkdir landscape && cd $_ && git clone https://github.com/gardener/garden-setup.git crop`
8. Save your local kubeconfig from the earlier steps into the `landscape` directory. Yes, you need a local copy; you cannot simply reference the existing one. See below.
9. Create a file named `acre.yml` in `landscape/` (**not** in `crop/`). See below for details.
10. Gardener cannot work with the kubeconfig that launches the gcloud auth-provider, so convert the kubeconfig to use a Kubernetes Service Account. See below.
11. Run `sow order -A` to see what order `sow` will apply things. It should return with an exit code of `0`.
12. Run `sow deploy -A`
13. Wait. Make a nice hot drink.

## detailed notes

### Deploying autoscaler CRDs

Deploying the autoscaler CRDs:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/940e304633c1ea68852672e39318b419ac9e155c/vertical-pod-autoscaler/deploy/vpa-v1-crd.yaml
```

### getting a landscape kubeconfig

There are two ways to do this.

* extract it from your existing kubeconfig
* use gcloud to get the kubeconfig and save it

To extract from your existing kubeconfig, assuming the context already is set:

```
kubectl config view --minify --raw > kubeconfig
```

To get it from gcloud:

```
KUBECONFIG=./kubeconfig gcloud container clusters get-credentials <your_cluster>
```

### acre.yml

You need to create a file named `acre.yml` in `landscape/`. Be **sure** not to make it in `landscape/crop/`,
where one already exists and must be left alone.

The reference for `acre.yaml` is [here](https://github.com/gardener/garden-setup#configuration-file-acreyaml).

The sample to be used is in this repository.

Important fields to note:

* `landscape.name` - the unique name for the gardener. Not only must this be unique in your project, but the name of the `etcd` GCS backup bucket with be `<landscape.name>-etcd-backup`. Bucket namnes **must** be unique globally, so your name must not already exist. Additionally, this must qualify as a DNS-1123 label, i.e. just alphanumeric characters and `-`. The restriction may be relaxed somewhat soon to allow any valid DNS name.
* `landscape.domain` - must be distinct from the cluster itself; will be used to create DNS entries for access to the Gardener API and UI. This must be a subdomain of a managed domain. E.g. if the managed domain is `abc.com`, then this field should be `something.abc.com`
* `landscape.cluster.kubeconfig` - relative path to the kubeconfig you created above, relative to the `landscape/` directory
* `landscape.networks` - CIDR ranges for the nodes, pods and services for the cluster you already deployed
* `landscape.iaas` - you can define several seeds here. For now, just define 1, which should be identical to the configuration of your base cluster
  * `landscape.iaas.credentials` - for google cloud, put in GKE service account credentials. See below.
* `landscape.dns` - information for managing DNS, as configured in `landscape.domain`. If this section is missing, it will try to use the managed DNS provider and credentials for the first `landscape.iaas` entry. If that type doesn't support managed DNS, it will fail.

### Google Cloud Service Account

Gardener requires a [Google Cloud Service Account](https://cloud.google.com/iam/docs/service-accounts) in order to manage things. That should have full rights over:

* GKE
* google cloud-dns
* gcs

Follow the instructions for setting it up, then create a key in json format, and save it to the appropriate location.

### GKE Service Account Credentials

Gardener requires a `kubeconfig` to manage each cluster in `landscape.iaas[*]`. When working with GKE, the `kubeconfig` provided, for example, by `gcloud container clusters get-credentials <cluster>` uses credentials that depend upon using the `gcloud` binary every time. This will not work for the credentials Gardener needs.

Instead, we set up a Kubernetes service account in the cluster (Note: Kubernetes service account, not Google Cloud service account), and then use its credentials. We use the `sa.yml` in this directory.

1. `kubectl apply -f sa.yml`
2. Get the secret name for the service account you just created: `KUBECONFIG=./kubeconfig kubectl -n kube-system get serviceAccounts gardener-admin -o jsonpath='{.secrets[0].name}'`
3. Get the token for that secret: `KUBECONFIG=./kubeconfig kubectl -n kube-system get secrets -o go-template='{{.data.token | base64decode}}'  <secret>`
4. Get the name of the first user: `KUBECONFIG=./kubeconfig kubectl config view -ojsonpath='{.users[0].name}'`. **Note:** This assumes you have a single user in your kubeconfig, per the steps above. If not, you will need to inspect it to find the right name for the user.
5. Update the kubeconfig to remove the auth-provider: `KUBECONFIG=./kubeconfig kubectl config unset users.<user>.auth-provider`
6. Update the kubeconfig to add the token to the user: `KUBECONFIG=./kubeconfig kubectl config set-credentials <user> --token=<token>`

Optionally, you can simplify steps 2 through 5 above with the following:

```
export KUBECONFIG=./kubeconfig
token=$(kubectl -n kube-system get secrets -oyaml -o jsonpath='{.data.token}' $(kubectl -n kube-system get serviceAccounts gardener-admin -o jsonpath='{.secrets[0].name}'))
user=$(kubectl config view -ojsonpath='{.users[0].name}')
kubectl config unset users.$(user).auth-provider`
kubectl config set-credentials $(user) --token=$(token)
```

