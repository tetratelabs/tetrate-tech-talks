# [Install MultiCluster](https://istio.io/latest/docs/setup/install/multicluster/)

More specifically, [Install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/).

## Create two clusters

1. Primary, name "primary-cluster"

    ```shell
    gcloud container clusters create primary-cluster \
      --cluster-version latest \
      --machine-type "n1-standard-2" \
      --num-nodes "3" \
      --network "default" \
      --zone us-central1-a
    ```

1. Remote, name "remote-cluster"

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
```

```shell
ALL_CLUSTER_NETTAGS=$(gcloud compute instances list --format='value(tags.items.[0])' | sort | uniq)
ALL_CLUSTER_NETTAGS=$(join_by , $(echo "${ALL_CLUSTER_NETTAGS}"))
```

```shell
gcloud compute firewall-rules create istio-mesh-internal \
  --allow=tcp,udp,icmp,esp,ah,sctp \
  --direction=INGRESS \
  --priority=900 \
  --source-ranges="${ALL_CLUSTER_CIDRS}" \
  --target-tags="${ALL_CLUSTER_NETTAGS}" --quiet
```


## [Create the CA certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/)

1. Per the instructions, create the root ca.

    ```shell
    make -f ../tools/certs/Makefile.selfsigned.mk root-ca
    ```

1. Create the intermediate certs for each of the two clusters:

    ```shell
    make -f ../tools/certs/Makefile.selfsigned.mk primary-cluster-cacerts
    ```

    And:

    ```shell
    make -f ../tools/certs/Makefile.selfsigned.mk remote-cluster-cacerts
    ```

1. Per the instructions, create the `cacerts` secret in each cluster.

    ```shell
    kubectl create secret generic cacerts -n istio-system \
          --from-file=primary-cluster/ca-cert.pem \
          --from-file=primary-cluster/ca-key.pem \
          --from-file=primary-cluster/root-cert.pem \
          --from-file=primary-cluster/cert-chain.pem
    ```

    And:

    ```shell
    kubectl create secret generic cacerts -n istio-system \
          --from-file=remote-cluster/ca-cert.pem \
          --from-file=remote-cluster/ca-key.pem \
          --from-file=remote-cluster/root-cert.pem \
          --from-file=remote-cluster/cert-chain.pem
    ```


## Construct the mesh

The document [Install Primary-Remote](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/) details the steps for constructing this mesh, which entails:

1. Install Istio on the primary cluster
1. Configure the east-west gateway
1. Expose istiod to the remote cluster
1. Enable Kube API server access in the remote cluster
1. Install Istio on the remote cluster

The [topology diagram](https://istio.io/latest/docs/setup/install/multicluster/primary-remote/arch.svg) will serve as a supporting reference as we build out the mesh.

Below is the adaptation of the steps to the specific cluster names used in this exercise.

## Steps

Define environment variables for each of the two Kubernetes contexts:

```shell
export CTX_PRIMARY=gke_eitan-tetrate_us-central1-a_primary-cluster
export CTX_REMOTE=gke_eitan-tetrate_us-central1-b_remote-cluster
```

Generate the Istio installation manifest file with the IstioOperator custom resource:

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
istioctl install --context="${CTX_PRIMARY}" -f primary-cluster.yaml
```

Install the east-west gateway:

```shell
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster primary-cluster --network network1 | \
    istioctl --context="${CTX_PRIMARY}" install -y -f -
```

```shell
kubectl --context="${CTX_PRIMARY}" get svc istio-eastwestgateway -n istio-system
```

Expose the control plane to cluster 2:

```shell
kubectl apply --context="${CTX_PRIMARY}" -n istio-system -f \
    samples/multicluster/expose-istiod.yaml
```

Allow the control plane to communicate with the kube api server on the remote cluster:

```shell
istioctl x create-remote-secret \
    --context="${CTX_REMOTE}" \
    --name=remote-cluster | \
    kubectl apply -f - --context="${CTX_PRIMARY}"
```

Install Istio on the remote cluster, configured to talk to the control plane on the primary:

```shell
export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_PRIMARY}" \
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
istioctl install --context="${CTX_REMOTE}" -f remote-cluster.yaml
```

## [Verify the installation](https://istio.io/latest/docs/setup/install/multicluster/verify/)

Follow the instructions on the page to:

- Deploy helloworld v1 to the primary cluster and v2 to the remote cluster.
- Deploy the sleep pod to both clusters.

These environment variables will help:

```shell
export CTX_CLUSTER1=gke_eitan-tetrate_us-central1-a_primary-cluster
export CTX_CLUSTER2=gke_eitan-tetrate_us-central1-b_remote-cluster
```



## Explore [locality load balancing](https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/)

Want clients to favor instances located in the same region and zone, but to fail over to instances in other zones and regions.

Configure a [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule) in the primary cluster:

```shell
kubectl --context="${CTX_PRIMARY}" apply -n sample -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.sample.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        maxRequestsPerConnection: 1
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
EOF
```

Next, call the `helloworld` service from the sleep pod on the primary cluster and see it favor the local instance (v1).

### Test failover

1. Drain the listeners on the sidecar of the `helloworld` pod in the primary zone:

    ```shell
    kubectl --context="${CTX_PRIMARY}" exec \
      "$(kubectl get pod --context="${CTX_PRIMARY}" -n sample -l app=helloworld \
      -l version=v1 -o jsonpath='{.items[0].metadata.name}')" \
      -n sample -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners
    ```

1. Make calls to the helloworld service from the sleep pod on the primary cluster once more, and watch it fail over to the endpoint running on the remote cluster (v2).
