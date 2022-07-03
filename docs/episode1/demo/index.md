# Connect VM Workloads to Istio mesh

This document is a recipe illustrating Istio mesh expansion using a single network and a single cluster.

We install Istio and deploy all [BookInfo](https://istio.io/latest/docs/examples/bookinfo/) services to the mesh, with the exception of the ratings service, which will run separately on a VM.

The idea is to make this work, and thereby to demonstrate that Istio supports a mesh where some services run in-cluster and some outside it.

The artifacts referenced in these instructions can be obtained from the GitHub repository for this site.

## Prerequisites

- A GCP or other cloud account

## Create K8s Cluster

```shell
./scripts/make-gke-cluster
```

Wait until cluster is ready.

## Create the VM

```shell
gcloud compute instances create my-mesh-vm --tags=mesh-vm \
  --machine-type=n1-standard-2 \
  --network=default --subnet=default \
  --image-project=ubuntu-os-cloud \
  --image=ubuntu-2110-impish-v20220309
```

## Install ratings app on the VM

Wait for the machine to be ready.

1. Copy over the ratings app

    ```shell
    gcloud compute scp --recurse bookinfo/ratings ubuntu@my-mesh-vm:ratings
    ```

1. ssh onto the VM

    ```shell
    gcloud compute ssh ubuntu@my-mesh-vm
    ```

1. Install nodejs, the ratings app and start it, test it.

    ```shell
    sudo apt-get update
    ```

    ```shell
    sudo apt-get install nodejs npm
    ```

1. Install dependencies

    ```shell
    cd ratings/
    npm install
    ```

1. Run the app:

    ```shell
    node ratings.js 9080 &
    ```

1. Test the app.

    Retrieve a rating.

    ```shell
    curl http://localhost:9080/ratings/123
    ```

### Allow POD-to-VM traffic on port 9080

```shell
CLUSTER_POD_CIDR=$(gcloud container clusters describe my-istio-cluster --format=json | jq -r '.clusterIpv4Cidr')
```

```shell
gcloud compute firewall-rules create "cluster-pods-to-vm" \
  --source-ranges=$CLUSTER_POD_CIDR \
  --target-tags=mesh-vm \
  --action=allow \
  --rules=tcp:9080
```

## Install Istio

```shell
istioctl install \
  --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION=true \
  --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_HEALTHCHECKS=true
```

## Deploy BookInfo (sans ratings)

1. Turn on sidecar-injection.

    ```shell
    k label ns default istio-injection=enabled
    ```

1. Deploy the reviews service.

    ```shell
    k apply -f bookinfo/bookinfo-reviews.yaml
    ```

    Important: the reviews service uses an environment variable named `SERVICES_DOMAIN` that we use to adjust the ratings app target url to reflect the fact that it resides in a different namespace.

1. Deploy the remaining services.

    ```shell
    k apply -f bookinfo/bookinfo-rest.yaml
    ```

## Install east-west gateway and expose Istiod

Control plane traffic between the VM and istiod goes through this gateway (see [the Istio documentation](https://istio.io/latest/docs/ops/deployment/vm-architecture/)).

1. Install the gateway

    ```shell
    ./scripts/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
    ```

1. Expose istiod

    ```shell
    k apply -n istio-system -f ./artifacts/expose-istiod.yaml
    ```

## Create the ratings namespace and service account

The ratings service running on the VM will map to the ratings namespace in kubernetes.

```shell
k create namespace ratings
```

```shell
k create serviceaccount bookinfo-ratings -n ratings
```

## Create the WorkloadGroup

A WorkloadGroup is a template for WorkloadEntry objects, see the [Istio reference](https://istio.io/latest/docs/reference/config/networking/workload-group/).

```shell
istioctl x workload group create \
  --name "ratings" \
  --namespace "ratings" \
  --labels app="ratings" \
  --serviceAccount "bookinfo-ratings" > workloadgroup.yaml
```

Apply the workloadgroup:

```shell
k apply -f workloadgroup.yaml -n ratings
```

## Generate VM artifacts

```shell
istioctl x workload entry configure -f workloadgroup.yaml -o vm_files --autoregister
```

Note: check that `vm_files/hosts` is not blank. If it is, it means you ran the command too soon.  Re-run it.

## VM configuration recipe

Copy the generated artifacts to the VM.

```shell
gcloud compute scp vm_files/* ubuntu@my-mesh-vm:
```

Ssh onto the VM

```shell
gcloud compute ssh ubuntu@my-mesh-vm
```

And, on the VM, run the following commands (taken from [here](https://istio.io/latest/docs/setup/install/virtual-machine/#configure-the-virtual-machine)).

```
sudo mkdir -p /etc/certs
sudo cp ~/root-cert.pem /etc/certs/root-cert.pem
sudo  mkdir -p /var/run/secrets/tokens
sudo cp ~/istio-token /var/run/secrets/tokens/istio-token
curl -LO https://storage.googleapis.com/istio-release/releases/1.13.2/deb/istio-sidecar.deb
sudo dpkg -i istio-sidecar.deb
sudo cp ~/cluster.env /var/lib/istio/envoy/cluster.env
sudo cp ~/mesh.yaml /etc/istio/config/mesh
sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
sudo mkdir -p /etc/istio/proxy
sudo chown -R istio-proxy /etc/certs /var/run/secrets /var/lib/istio /etc/istio/config /etc/istio/proxy
```

## Exercise 1

Watch the WorkloadEntry get created as a consequence of the VM registering with the mesh.

```shell
k get workloadentry -n ratings -w
```

On the VM:

```shell
sudo systemctl start istio
```

Notice the workload entry show up in the listing.  This can take up to a minute.


## Exercise 2

Although the ratings services does not need to call back into the mesh, we can manually test communication from the VM into the mesh.

From the VM, run:

```shell
curl details.default.svc:9080/details/123
```

## Exercise 3

Test communication from a pod to the ratings service running on the VM.

Create a ClusterIP service to front the application:

```shell
k apply -n ratings -f bookinfo/bookinfo-ratings-service.yaml
```

Create a temporary client pod in the default namespace

```shell
k run curlpod --image=radial/busyboxplus:curl -it --rm
```

From within the container, run the curl command:

```shell
curl ratings.ratings:9080/ratings/123
```

Finally, `exit` the container.


## Put it all together

Expose BookInfo:

```shell
k apply -f bookinfo/bookinfo-gateway.yaml
```

Grab your load balancer public IP address:

```shell
GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Open a browser and visit the BookInfo product page (at /productpage).  Verify that you can see ratings on the page.

```shell
curl $GATEWAY_IP/productpage
```

## References

- [Istio VM Architecture](https://istio.io/latest/docs/ops/deployment/vm-architecture/)
- [Basic recipe from the Istio docs](https://istio.io/latest/docs/setup/install/virtual-machine/)
- [GCP-specific information on mesh expansion](https://github.com/GoogleCloudPlatform/istio-samples/tree/master/mesh-expansion-gce) (dated)
