# [Install MultiCluster](https://istio.io/latest/docs/setup/install/multicluster/)

More specifically, [Install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/).

## Create the two clusters:

1. primary, name "primary-cluster"

    ```shell
    gcloud container clusters create primary-cluster \
      --cluster-version latest \
      --machine-type "n1-standard-2" \
      --num-nodes "3" \
      --network "default" \
      --zone us-central1-a
    ```

1. remote, name "remote-cluster"

    ```shell
    gcloud container clusters create remote-cluster \
      --cluster-version latest \
      --machine-type "n1-standard-2" \
      --num-nodes "3" \
      --network "default" \
      --zone us-central1-b
    ```

## Allow cluster-to-cluster communication

See [reference](https://github.com/GoogleCloudPlatform/istio-samples/tree/master/multicluster-gke/single-control-plane):

```shell
function join_by { local IFS="$1"; shift; echo "$*"; }

ALL_CLUSTER_CIDRS=$(gcloud container clusters list --format='value(clusterIpv4Cidr)' | sort | uniq)
ALL_CLUSTER_CIDRS=$(join_by , $(echo "${ALL_CLUSTER_CIDRS}"))
ALL_CLUSTER_NETTAGS=$(gcloud compute instances list --format='value(tags.items.[0])' | sort | uniq)
ALL_CLUSTER_NETTAGS=$(join_by , $(echo "${ALL_CLUSTER_NETTAGS}"))

gcloud compute firewall-rules create istio-multicluster-test-pods \
  --allow=tcp,udp,icmp,esp,ah,sctp \
  --direction=INGRESS \
  --priority=900 \
  --source-ranges="${ALL_CLUSTER_CIDRS}" \
  --target-tags="${ALL_CLUSTER_NETTAGS}" --quiet
```


## [Create the CA certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/)

1. Per the instructions, create the root ca.

1. Create the intermediate certs for each of the two clusters:

    ```shell
    make -f ../tools/certs/Makefile.selfsigned.mk primary-cluster-cacerts
    ```

    And:

    ```shell
    make -f ../tools/certs/Makefile.selfsigned.mk remote-cluster-cacerts
    ```

1. Per the instructions, create the `cacerts` secret in each cluster.

## Install Istio et al

The main instructions to [install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/) provide the next steps:

1. Install Istio on the primary cluster
1. Configure the east-west gateway
1. Expose istiod
1. Enable API server access to the remote cluster
1. Install Istio on the remote cluster

Below is the adaptation of those steps to the specific cluster names used in this exercise.

## Steps

Define environment variables for each of the two Kubernetes contexts:

```shell
export CTX_CLUSTER1=gke_eitan-tetrate_us-central1-a_primary-cluster
export CTX_CLUSTER2=gke_eitan-tetrate_us-central1-b_remote-cluster
```

Generate the istio installation manifest file with the IstioOperator custom resource:

```shell
cat <<EOF > primary-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: primary-cluster
      network: network1
EOF
```

Install Istio on the primary cluster:

```shell
istioctl install --context="${CTX_CLUSTER1}" -f primary-cluster.yaml
```

Install the east-west gateway:

```shell
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster primary-cluster --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -
```

```shell
kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
```

Expose the control plane to cluster 2:

```shell
kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f \
    samples/multicluster/expose-istiod.yaml
```

Allow the control plane to communicate with the kube api server on the remote cluster:

```shell
istioctl x create-remote-secret \
    --context="${CTX_CLUSTER2}" \
    --name=remote-cluster | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
```

Install Istio on the remote cluster, configured to talk to the control plane on the primary:

```shell
export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_CLUSTER1}" \
    -n istio-system get svc istio-eastwestgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

```shell
cat <<EOF > remote-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: remote-cluster
      network: network1
      remotePilotAddress: ${DISCOVERY_ADDRESS}
EOF
```

And finally:

```shell
istioctl install --context="${CTX_CLUSTER2}" -f remote-cluster.yaml
```

## [Verify the installation](https://istio.io/latest/docs/setup/install/multicluster/verify/)

Follow the instructions on the page to:

- Deploy helloworld v1 to the primary cluster and v2 to the remote cluster.
- Deploy the sleep pod to both clusters.

