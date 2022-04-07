# Upgrading Istio

This document demonstrates upgrading Istio in a way that allows operators to be in control of the transition of workloads from one version to the next.

This document is an adaptation of the document entitled [Canary Upgrades](https://istio.io/latest/docs/setup/upgrade/canary/) from the official Istio documentation.

The [BookInfo](https://istio.io/latest/docs/examples/bookinfo/) sample application will serve as the workload under test.

!!! warning

    [Feature status](https://istio.io/latest/docs/releases/feature-stages/#core) of Revision Based Upgrades is **alpha**
    

In this exercise, we will:

- Install a Kubernetes cluster
- Install Istio v1.12.5
- Deploy the BookInfo sample application to the `default` namespace.

Next, we:

- Install Istio v1.13.2, triggering solely the upgrade of the ingress gateway (not the workloads)
- Demonstrate namespace labeling as the mechanism for controlling the desired version of Istio to use for workloads in that namespace.
- Teardown previous version of Istio, completing the upgrade.

Finally, we will demonstrate an alternative to labeling namespaces directly, by using what Istio calls [Revision Tags](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-tag).

!!! note

    In the instructions that follow, I use GCP as my infastructure, but feel free to use any Kubernetes you like.

## Create a K8s Cluster

```shell
gcloud container clusters create my-istio-cluster \
  --cluster-version latest \
  --machine-type "n1-standard-2" \
  --num-nodes "3" \
  --network "default"
```

Wait until cluster is ready.

## Client Setup

In a workspace directory of your choice, download two Istio releases, 1.12.5 and 1.13.2.

Be sure to use the hardware architecture matching your workstation:

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.12.5 TARGET_ARCH=arm64 sh -
```

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.2 TARGET_ARCH=arm64 sh -
```

### Setting your PATH

I use [direnv](https://direnv.net/) to easily update my PATH so that istioctl points to version 1.12.5 or version 1.13.2 as a function of the directory that i navigate to.

Given a file named `.envrc` in the folder `istio-1.12.5`:

```shell
#!/bin/sh

export PATH=./bin:$PATH
```

I can run `direnv allow`, and note that running `istioctl version` from that directory returns `1.12.5`.

Likewise, the same file in the `istio-1.13.2` directory allows me to quickly switch to that version.


## Install Istio

[Pre-check](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-experimental-precheck):

```shell
istioctl x precheck
```

Install:

```shell
istioctl install --set revision=1-12-5
```

Check:

```shell
k get pods -n istio-system -l app=istiod
k get svc -n istio-system -l app=istiod
k get mutatingwebhookconfigurations
```

Check the ingressgateway Istio version:

```shell
istioctl proxy-status
```

## Label the default namespace

```shell
k label namespace default istio.io/rev=1-12-5
```

Check:

```shell
k get ns -Listio.io/rev
```

## Deploy BookInfo

```shell
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

Check:

```shell
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

Configure ingress:

```shell
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

Check:

```shell
GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Open a browser and visit the BookInfo product page (at `/productpage`).

```shell
curl $GATEWAY_IP/productpage
```


## Verifying that workloads use version 1.12.5

1. With `istioctl proxy-status`

    ```shell
    istioctl proxy-status
    ```

1. Checking the server_info endpoint of a sidecar:

    ```shell
    istioctl dashboard envoy deployment/details-v1.default
    ```

    And check that the ANNOTATIONS revision is 1-12-5

1. Describe the pod

    ```shell
    k describe pod details-v1-<tab>
    ```

## Switch directories

```shell
cd ../istio-1.13.2
```

Make sure your PATH now points to version 1.13.2 of the `istioctl` CLI:

```shell
istioctl version
```

## Install Istio version 1.13.2

Pre-check:

```shell
istioctl x precheck
```

[Analyze](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/)?:

```shell
istioctl analyze
```

Also, see the `istioctl analyze` [command reference](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-analyze), and
[configuration analysis messsages](https://istio.io/latest/docs/reference/config/analysis/).  Possible [false positive](https://github.com/istio/istio/issues/22698) and [Gateway resource](https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway).


Install:

```shell
istioctl install --set revision=1-13-2
```

- Both 1-13-2 and 1-12-5 istiod's are installed
- ingressgateway has been upgraded to 1.13.2
- Pods still use 1-12-5

Check:

```shell
k get pods -n istio-system -l app=istiod
```

And:

```shell
istioctl proxy-status
```

## Update namespace label

```shell
kubectl label namespace default istio.io/rev=1-13-2 --overwrite
```

Check:

```shell
k get ns -Listio.io/rev
```

## Restart the deployments in the default namespace

```shell
kubectl rollout restart deployment -n default
```

```shell
istioctl proxy-status
```

## Teardown the old version

```shell
istioctl x uninstall --revision 1-12-5
```

Verify that the bookinfo application is still alive and well.

## Using Revision Tags

Instead of specifying the actual revision we want, we use a semantic name, like `prod`:

```shell
k label ns default istio.io/rev=prod --overwrite
```

And then associate prod to a revision:

```shell
istioctl tag set prod --revision 1-13-2
```

Check:

```shell
istioctl tag list
```

To change what prod means, we associate a different revision to it:

```shell
istioctl tag set prod --revision 1-12-5 --overwrite
```

## Closing thoughts

- Operators can make available a new version of Istio and notify developers to update their workloads on their own time
- Develop a process, and automation for implementing Istio ugprades, and rollbacks.
  Start by listening to Pratima Nambiar from Salesforce in her [Istio Community Talk](https://youtu.be/j273hsoqza0?t=1308).

## References

- [Canary upgrades](https://istio.io/latest/docs/setup/upgrade/canary/)
- [Revision tags](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-tag)
- [Blog entry on revision tags](https://istio.io/latest/blog/2021/revision-tags/)