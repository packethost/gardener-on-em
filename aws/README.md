# Gardener - Deploying a "garden" cluster on AWS

This guide deploys a Gardener base cluster (a "garden" cluster) on AWS. The guide uses
[kops](https://kops.sigs.k8s.io/) to bootstrap the Kubernetes cluster on top of which the garden
cluster is deployed.

The following instructions assume a Linux workstation. The instructions for macOS should be very
similar with the exception of the paths used for downloading tools.

## Requirements

- `kubectl`
- A Route53 hosted zone (more info [here](https://kops.sigs.k8s.io/getting_started/aws/#configure-dns))
- An IAM user to be used by Gardener with the following permissions:
  - Full access to Route53
  - Full access to VPC (required only for deploying AWS workload clusters)
  - Full access to EC2 (required only for deploying AWS workload clusters)
  - Full access to IAM (required only for deploying AWS workload clusters)
- An S3 bucket for storing the kops cluster state
- An ssh key pair for node access

## Instructions

### Install `kops`

Download and install the `kops` binary:

```
curl -Lo kops \
  "https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest \
  | grep tag_name \
  | cut -d '"' -f 4)/kops-linux-amd64"
chmod +x kops
sudo mv kops /usr/local/bin/kops
```

### Create a cluster

Create a directory for the cluster- and Gardener-related files and navigate to it:

```
mkdir aws-garden && cd $_
```

Set the following environment variables:

```
export NAME=my-garden.example.com           # The name of the cluster as an FQDN
export KOPS_STATE_STORE=s3://my-kops-bucket # An S3 path to store the cluster state in
export SSH_PUBKEY=~/.ssh/my-key.pub         # An SSH public key to authorize on the nodes
```

Run the following command to generate a cluster configuration:

```
kops create cluster \
  --zones=eu-central-1a,eu-central-1b,eu-central-1c \
  --node-count 7 \
  --node-size t3a.large \
  --network-cidr 172.17.0.0/16 \
  --ssh-public-key $SSH_PUBKEY \
  --dry-run \
  --output yaml > cluster.yaml \
  $NAME
```

The default CIDRs used by kops for pods and services collide with some Gardener defaults. Edit
`cluster.yaml` by running the following:

```
sed -i 's/^\(\s\snonMasqueradeCIDR:\).*/\1 100.64.0.0\/10/' cluster.yaml
sed -i '/^\s\snonMasqueradeCIDR:/a \  podCIDR: 100.96.0.0/11\n\  serviceClusterIPRange: 100.64.0.0/13' cluster.yaml
```

Verify the CIDR configuration:

```
cat cluster.yaml | grep -A 5 networkCIDR
```

Sample output:

```
  networkCIDR: 172.17.0.0/16
  networking:
    kubenet: {}
  nonMasqueradeCIDR: 100.64.0.0/10
  podCIDR: 100.96.0.0/11
  serviceClusterIPRange: 100.64.0.0/13
```

Deploy the cluster:

```
kops create cluster \
  --zones=eu-central-1a,eu-central-1b,eu-central-1c \
  --node-count 7 \
  --node-size t3a.large \
  --network-cidr 172.17.0.0/16 \
  --ssh-public-key $SSH_PUBKEY \
  --config cluster.yaml \
  $NAME \
  --yes
```

Run the following command and wait for the cluster to bootstrap:

```
kops validate cluster --wait 10m
```

Verify connectivity with the cluster:

```
kubectl get nodes
```

Sample output:

```
NAME                                              STATUS   ROLES    AGE     VERSION
ip-172-17-100-182.eu-central-1.compute.internal   Ready    node     2m21s   v1.18.12
ip-172-17-122-29.eu-central-1.compute.internal    Ready    node     2m16s   v1.18.12
ip-172-17-44-222.eu-central-1.compute.internal    Ready    master   4m40s   v1.18.12
ip-172-17-58-41.eu-central-1.compute.internal     Ready    node     2m12s   v1.18.12
ip-172-17-81-234.eu-central-1.compute.internal    Ready    node     2m17s   v1.18.12
ip-172-17-92-16.eu-central-1.compute.internal     Ready    node     2m26s   v1.18.12
```

### Deploy the Vertical Pod Autoscaler

The garden cluster expects CRDs belonging to the
[Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
to exist on the cluster.

Deploy the VPA CRDs:

```
# TODO: Should we pin a specific version?
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/940e304633c1ea68852672e39318b419ac9e155c/vertical-pod-autoscaler/deploy/vpa-v1-crd.yaml
```

### Install `sow`

Gardener uses a utility named `sow` for bootstrapping a garden cluster.

Download `sow` and add the binary to your `PATH`:

```
git clone "https://github.com/gardener/sow"
export PATH=$PATH:$PWD/sow/docker/bin
```

Verify `sow` works:

```
sow version
```

Sample output:

```
sow version 3.3.0-dev
```

### Prepare a "landscape" directory

Gardener uses the term "landscape" to refer to an instance of the Gardener stack. We need to
prepare a directory which would contain the necessary files for deploying Gardener.

Create a landscape directory:

```
mkdir landscape && cd $_
```

Gardener uses a tool called `garden-setup` to bootstrap the garden cluster. Clone the
`garden-setup` repository into a directory called `crop` inside the `landscape` directory:

```
# TODO: Should we pin a specific version?
git clone "https://github.com/gardener/garden-setup" crop
```

Generate a kubeconfig file which Gardener can use to bootstrap the garden cluster:

```
kops export kubecfg --kubeconfig ./kubeconfig
```

Verify the generated kubeconfig file works:

```
KUBECONFIG=./kubeconfig kubectl get pods -A
```

Sample output:

```
NAMESPACE     NAME                                                                     READY   STATUS    RESTARTS   AGE
kube-system   dns-controller-8554fd9c56-zlhbz                                          1/1     Running   0          9m24s
kube-system   etcd-manager-events-ip-172-17-44-222.eu-central-1.compute.internal       1/1     Running   0          8m46s
kube-system   etcd-manager-main-ip-172-17-44-222.eu-central-1.compute.internal         1/1     Running   0          8m46s
kube-system   kops-controller-9ghxx                                                    1/1     Running   0          8m35s
kube-system   kube-apiserver-ip-172-17-44-222.eu-central-1.compute.internal            2/2     Running   1          7m48s
kube-system   kube-controller-manager-ip-172-17-44-222.eu-central-1.compute.internal   1/1     Running   0          8m56s
kube-system   kube-dns-6c699b5445-db7n6                                                3/3     Running   0          9m24s
kube-system   kube-dns-6c699b5445-tr9h2                                                3/3     Running   0          6m56s
kube-system   kube-dns-autoscaler-cd7778b7b-2k5lk                                      1/1     Running   0          9m24s
kube-system   kube-proxy-ip-172-17-100-182.eu-central-1.compute.internal               1/1     Running   0          5m39s
kube-system   kube-proxy-ip-172-17-122-29.eu-central-1.compute.internal                1/1     Running   0          6m32s
kube-system   kube-proxy-ip-172-17-44-222.eu-central-1.compute.internal                1/1     Running   0          7m34s
kube-system   kube-proxy-ip-172-17-58-41.eu-central-1.compute.internal                 1/1     Running   0          6m52s
kube-system   kube-proxy-ip-172-17-81-234.eu-central-1.compute.internal                1/1     Running   0          6m26s
kube-system   kube-proxy-ip-172-17-92-16.eu-central-1.compute.internal                 1/1     Running   0          6m38s
kube-system   kube-scheduler-ip-172-17-44-222.eu-central-1.compute.internal            1/1     Running   0          8m54s
```

Create a file named `acre.yaml` and edit the fields marked with "change me" comments:

```
cat <<EOF >acre.yaml
landscape:
  name: my-gardener # change me
  # Used to create endpoints for the Gardener API and UI. Do *NOT* use the same domain as the one
  # used for creating the k8s cluster.
  domain: my-gardener.example.com # change me

  cluster:
    kubeconfig: ./kubeconfig
    networks:
      nodes: 172.17.0.0/16
      pods: 100.96.0.0/11
      services: 100.64.0.0/13

  iaas:
    - name: (( iaas[0].type ))
      type: aws
      shootDefaultNetworks:
        pods: 10.96.0.0/11
        services: 10.64.0.0/13
      region: eu-central-1
      zones:
        - eu-central-1a
        - eu-central-1b
        - eu-central-1c
      seedConfig:
        backup:
          active: false
      credentials:
        # Used by Gardener to create Route53 DNS records.
        accessKeyID: AKIAxxxxxxxx         # change me
        secretAccessKey: xxxxxxxxxxxxxxxx # change me

  etcd:
    backup:
      active: false
      type: s3
      region: (( iaas[0].region ))
      credentials: (( iaas[0].credentials ))

  dns:
    type: aws-route53
    credentials: (( iaas[0].credentials ))

  identity:
    users:
      # Used for logging into the Gardener UI.
      - email: "someone@example.com" # change me
        username: "someone"          # change me
        password: "securepassword"   # change me
EOF
```

Additional fields can be changed as necessary. For more information about the configuration scheme
visit the
[reference](https://github.com/gardener/garden-setup/tree/9bc9ecc7e31eb444b0b04770277831be32ff0676#configuration-file-acreyaml).

### Deploy Gardener

To bootstrap a garden cluster on the EKS cluster, run the following commands inside the `landscape`
directory:

```
# Should produce no output.
sow order -A
```

```
sow deploy -A
```

This process can take around 10 minutes. When done, an output similar to the following is shown:

```
===================================================================
Dashboard URL -> https://gardener.ing.my-gardener.example.com
===================================================================
generating exports
exporting file dashboard_url
*** species dashboard deployed
```

Visit the URL in a browser and log in using the email and password which were specified in
`acre.yaml`.
