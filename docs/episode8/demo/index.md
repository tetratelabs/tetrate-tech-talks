# [Install MultiCluster](https://istio.io/latest/docs/setup/install/multicluster/)

More specifically, [Install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/).

## Create the two clusters:

1. primary, name "my-istio-cluster"

    ```shell
    gcloud container clusters create my-istio-cluster \
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
    make -f ../tools/certs/Makefile.selfsigned.mk my-istio-cluster-cacerts
    ```

    And:

    ```shell
    make -f ../tools/certs/Makefile.selfsigned.mk remote-cluster-cacerts
    ```

1. Per the instructions, create the `cacerts` secret in each cluster.

## Install Istio et al

Go back to the main instructions to [install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/).

Follow the steps to:

1. Install Istio on the primary cluster
1. Configure the east-west gateway
1. Expose istiod
1. Enable API server access to the remote cluster
1. Install Istio on the remote cluster

## [Verify the installation](https://istio.io/latest/docs/setup/install/multicluster/verify/)

Follow the instructions on the page to:

- deploy helloworld v1 to the primary cluster and v2 to the remote cluster.
- deploy the sleep pod to both clusters.