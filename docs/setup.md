# Setup

## A Cluster, Gateway CRDs, and Istio

Provision a Kubernetes cluster of your choosing.
This could be a local [kind](https://kind.sigs.k8s.io/) cluster, or one running in your favorite cloud environment.
My personal setup uses a local [k3d](https://k3d.io) cluster.

```shell
k3d cluster create my-cluster \
    --api-port 6443 \
    --k3s-arg "--disable=traefik@server:0" \
    --port 80:80@loadbalancer \
    --port 443:443@loadbalancer
```

Be sure to install the Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) CRDs:

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

[Install Istio in ambient mode](https://istio.io/latest/docs/ambient/install/) (I am using the stable version at time of writing, `1.26.2`):

```shell
istioctl install \
    --set profile=ambient \
    --set values.global.platform=k3d
```

## Provision a test client

Deploy a test client in the form of a pod running a container with `curl` installed:

```shell
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/curl/curl.yaml
```

Make workloads in the `default` namespace a part of the mesh:

```shell
kubectl label namespace default istio.io/dataplane-mode=ambient
```

Note that the client workload does not require a sidecar in order to be part of the "mesh":

```shell
kubectl get pod -n default
```

We also no longer need to concern ourselves with restarting workloads.

## Configure access logging

Enable access logging for Gateways and Waypoints:

```shell
kubectl apply -f - <<EOF
---
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: enable-access-logging
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: envoy
EOF
```

Above, we use the `istio-system` "root" namespace to ensure that the configuration applies to all workloads in the mesh (all namespaces).

With setup out of the way, we begin by deploying a set of microservices to our cluster, and making them a part of the mesh.

## Deploy a set of microservices

The canonical example used by the Istio project to represent a set of microservices is the [`bookinfo` sample application](https://istio.io/latest/docs/examples/bookinfo/).
`bookinfo` consists of the following microservices, all running on the port `9080`: `productpage`, `details`, `reviews` and `ratings`.

Begin by deploying these microservices to the Kubernetes cluster, in their own namespace, `bookinfo`:

```shell
kubectl create namespace bookinfo
```

```shell
kubectl apply -n bookinfo \
  -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo.yaml
```

Add all of the `bookinfo` workloads to the mesh, by [labeling their namespace](https://istio.io/latest/docs/ambient/getting-started/secure-and-visualize/):

```shell
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

Let us start with controlling ingress traffic flows.
