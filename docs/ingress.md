# N/S/E/W: The Power of a single API to secure, observe, and control API traffic in all directions

## Introduction

In the latest releases of Istio, with support for Ambient mode becoming Generally Available (GA), and with its support for the Kubernetes Gateway API, we are finally at a point where platform engineers can provision, configure and control network traffic in uniform way, and without the overhead of sidecars.

It is important to stress that the above term "network traffic" implies traffic in all directions: at ingress, at egress, as well as internal "mesh" traffic, also known as "east-west" traffic.
This vision of a single, uniform API to control traffic in all directions is something we at Solo.io have dubbed "Omni" (see [What is Omni?](https://www.solo.io/resources/video/what-is-omni-part-1-of-2)).

This document is a hands-on exploration of configuring ingress, egress, and mesh traffic, in order to see first-hand how this is done.
The main objective is to get a sense for how these different activities are performed with a single API, and where the process of configuring and controlling traffic is in many ways the same.

## Setup

Provision a Kubernetes cluster.

```shell
k3d cluster create my-cluster \
    --api-port 6443 \
    --k3s-arg "--disable=traefik@server:0" \
    --port 80:80@loadbalancer \
    --port 443:443@loadbalancer
```

Be sure to install the Kubernetes Gateway API CRDs:

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

Install Istio in ambient mode (I used the current stable version, `1.26.2`):

```shell
istioctl install \
    --set profile=ambient \
    --set values.global.platform=k3d
```

### Provision a test client

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

### Configure access logging

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

The canonical example used by the Istio project to represent a set of microservices is the `bookinfo` sample application.
`bookinfo` consists of the following microservices, all running on the port `9080`: `productpage`, `details`, `reviews` and `ratings`.

Begin by deploying these microservices to the Kubernetes cluster, in their own namespace, `bookinfo`:

```shell
kubectl create namespace `bookinfo`
```

```shell
kubectl apply -n bookinfo \
  -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo.yaml
```

We are no longer required to define "subsets", specific versions of a microservice, with a `DestinationRule`, we can simply use a Kubernetes `Service` definition with the right combination of selectors.

Apply the following resource, which defines these "sub" services:

```shell
kubectl apply -n bookinfo \
  -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo-versions.yaml
```

Finally, add all of the `bookinfo` workloads to the mesh:

```shell
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

Let us start with controlling ingress traffic flows.

## Configure & Control Ingress

As you review the steps (or recipe) outlined below, observe the general process we follow to provision gateways, configure traffic flows, and overlay authorization policies.

Expose `bookinfo`'s web UI, served by the `productpage` service, to the internet through an ingress gateway.

### Provision and configure the Gateway

We need to make some design decisions that pertain to our gateway:

- The gateway should accept HTTPS traffic.
- It should redirect HTTP traffic to the HTTPS scheme.
- The gateway should terminate TLS.

We also have the option to deploy either a dedicated gateway for `bookinfo`, or a shared gateway that routes ingress traffic on behalf of mulitple APIs and applications that we intend to deploy to our cluster.

Let us proceed with a shared gateway, that we decide to deploy to a namespace named `common-infrastructure`.

Create the namespace:

```shell
kubectl create namespace common-infrastructure
```

Make the namespace part of the mesh:

```shell
kubectl label namespace common-infrastructure istio.io/dataplane-mode=ambient
```

Thanks to the Kubernetes Gateway API, we can provision our gateway on demand, by applying the following `Gateway` resource:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: common-infrastructure
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
  - name: bookinfo-https
    protocol: HTTPS
    port: 443
    hostname: bookinfo.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: bookinfo-cert
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            self-serve-ingress: "true"
```

Let us break down the above configuration, we:

- Name the gateway `shared-gateway`, and place it in the `common-infrastructure` namespace.
- Associate the ingress gateway with Istio by specifying the `istio` gatewayClassName.
- Expose a listener on port 80 that we intend to configure to redirect traffic to port 443.
- Expose a listener on port 443, bound to the [ficticious] hostname `bookinfo.example.com`, configured to terminate TLS.
- Configure the gateway to allow the binding of routes that reside in namespaces labeled `self-serve-ingress` with value `true` (an adhoc convention).

Let us go ahead and apply that label to the `bookinfo` namespace since we know we will need to define a route there for directing requests to the `productpage` service:

```shell
kubectl label ns bookinfo self-serve-ingress=true
```

The generation of the TLS certificate for the hostname `bookinfo.example.com` and its storage into a kubernetes `Secret` by the name of `bookinfo-cert` can be performed using many different tools and methods, and is left as an exercise.

I used the [`step` CLI](https://smallstep.com/docs/step-cli/) to generate the certificate:

```shell
step certificate create bookinfo.example.com bookinfo.crt bookinfo.key \
  --profile self-signed --subtle --no-password --insecure
```

And this command to apply the secret to the cluster:

```shell
kubectl create secret tls bookinfo-cert -n common-infrastructure \
  --cert=bookinfo.crt --key=bookinfo.key
```

Applying the `Gateway` resource to the cluster triggers a number of actions:

1. A Gateway is provisioned:  Istio will provision an Envoy proxy for the gateway, and program it accordingly, as we just outlined.  You will see a deployment in the `common-infrastructure` namespace (which begets a ReplicaSet and in turn a Pod).
2. Istio will program the proxy according to the `Gateway` configuration specification we supplied.
3. A service of type LoadBalancer is created.  For a Kubernetes cluster deployed in the cloud, this implies the creation of an L4-type load balancer with a public static IP address, configured to route requests to the Envoy proxy deployment.  A DNS record should be configured for the hostname pointing to that public IP address.

At this point the Gateway can receive HTTP requests from the internet, but will return a 404, until we configure how we wish the proxy to route those requests.

### Controlling routing

The HTTPRoute resource to expose the `productpage` service's endpoints is bundled with Istio's `bookinfo` sample application:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-route
  namespace: bookinfo
spec:
  parentRefs:
  - name: shared-gateway
    namespace: common-infrastructure
    sectionName: bookinfo-https
  rules:
  - matches:
    - path:
        type: Exact
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: Exact
        value: /login
    - path:
        type: Exact
        value: /logout
    - path:
        type: PathPrefix
        value: /api/v1/products
    backendRefs:
    - name: productpage
      port: 9080
```

It is worth calling out how the route binds to the Gateway through the `parentRefs` field, and that we bind specifically to the port 443 listener:  when a request arrives over HTTPS, the routing rules will match.  We expose various endpoints including the `productpage` web UI, access to static resources, the `/login` and `/logout` endpoints, and the `productpage` API.  The `backendRef` directs matching requests to the service running inside the cluster.

The Gateway API offers many more controls over how requests get routed, including traffic splitting, request mirroring, support for timeouts and retries.  Istio has had its own resources for traffic management, namely the `VirtualService` resource, and today fully supports the Gateway API as the defacto standard.

The same `HTTPRoute` (and other Route-type resources) is also used for configuring east-west traffic, as we will see shortly.

### Add an AuthorizationPolicy

Istio allows us to overlay a security configuration as well.

For example, we can define an AuthorizationPolicy to allow only the Ingress gateway to call the `productpage` service.

```yaml
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
 name: allow-gateway-to-productpage
 namespace: bookinfo
spec:
 selector:
   matchLabels:
     app: productpage
 action: ALLOW
 rules:
 - from:
   - source:
       principals:
       - cluster.local/ns/common-infrastructure/sa/shared-gateway-istio
```

Above, note the [SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/) identity of the Gateway `shared-gateway` is a function of its namespace and associated service account name.

This authorization policy is not strictly an ingress-specific concern, as it applies to any requests targeting the `productpage` service.

### Test it

TODO: get the gw ip address, send in a curl request to the gw and see response from productpage service

Attempt to call `productpage` from the unauthorized `curl` client running in the `default` namespace:

```shell
kubectl exec deploy/curl -- curl -s productpage.bookinfo:9080/productpage
```

Response should indicate a terminated request:

```console
command terminated with exit code 6
```

For the details, we can inspect the logs of the ztunnel component:

```shell
kubectl logs --follow -n istio-system -l app.kubernetes.io/name=ztunnel
```

The output in so many words indicates a "401 Unauthorized" response:

```console
2025-06-30T06:41:34.685667Z     error   access  connection complete     src.addr=10.42.0.9:53890 src.workload="curl-5b549b49b8-sf5qz" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.42.0.17:15008 dst.hbone_addr=10.42.0.17:9080 dst.service="productpage.bookinfo.svc.cluster.local" dst.workload="productpage-v1-78dfd4688c-rbm5g" dst.namespace="bookinfo" dst.identity="spiffe://cluster.local/ns/bookinfo/sa/bookinfo-productpage" direction="inbound" bytes_sent=0 bytes_recv=0 duration="0ms" error="connection closed due to policy rejection: allow policies exist, but none allowed"
2025-06-30T06:41:34.685787Z     error   access  connection complete     src.addr=10.42.0.9:44984 src.workload="curl-5b549b49b8-sf5qz" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.42.0.17:15008 dst.hbone_addr=10.42.0.17:9080 dst.service="productpage.bookinfo.svc.cluster.local" dst.workload="productpage-v1-78dfd4688c-rbm5g" dst.namespace="bookinfo" dst.identity="spiffe://cluster.local/ns/bookinfo/sa/bookinfo-productpage" direction="outbound" bytes_sent=0 bytes_recv=0 duration="0ms" error="http status: 401 Unauthorized"
```

### Analysis for Ingress

Here is a general outline of the process we followed to configure ingress:

- Provision and configure the gateway with the `Gateway` resource
- Configure routing with the `HTTPRoute` (or a different protocol-specific route) resource
- Configure security policy

Let us next look at how we might configure and control egress traffic, and see how the process resembles what we just did for ingress.

## Configure & Control Egress

Below is a general example for configuring egress, taken from [the documentation](https://ambientmesh.io/docs/traffic/mesh-egress/).

### Provision a gateway

Just like we did for ingress, we begin by provisioning a Gateway.

This time we enlist the help of the Istio CLI, specifically the command `istioctl waypoint apply`:

```shell
istioctl waypoint apply --enroll-namespace --name egress-gateway --namespace common-infrastructure
```

The above command generate a Gateway resource and applies it to the indicated namespace.

We can verify this by asking for resources of type Gateway in the `common-infrastructure` namespace:

```shell
kubectl get gtw -n common-infrastructure
```

The output indicates that we now have two Gateways, though the GatewayClass type is different for each:

```console
NAME             CLASS            ADDRESS        PROGRAMMED   AGE
egress-gateway   istio-waypoint   10.43.227.68   True         4h3m
shared-gateway   istio            192.168.97.2   True         21m
```

The other subtle but important point in the above `istioctl` command is the `--enroll-namespace` flag:  any services defined in that namespace will be proxied by this egress gateway.

#### Create a ServiceEntry

The `ServiceEntry` resource is designed to allow Istio to be aware of services that reside outside the mesh.

Here is for example a `ServiceEntry` that communicates to Istio that mesh workloads may call the hostname `httpbin.org`.

```shell
kubectl apply -f - <<EOF
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin.org
  namespace: common-infrastructure
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
    targetPort: 443 # New: send traffic originally for port 80 to port 443
  resolution: DNS
EOF
```

In this case, the IP address of the target workload is obtained through DNS resolution.

The convention is that clients call `httpbin.org` using the url `http://httpbin.org/` on port 80.
The ServiceEntry however, specifies a `targetPort` of 443, indicating that the outbound request to the actual service will be on a different port.

Since the `ServiceEntry` is defined in the `common-infrastructure` namespace, traffic from clients targeting the host `httpbin.org` will be routed to the egress gateway.
Communication from the client to the gateway will use mutual TLS by virtue of both ends of that communication being a part of the mesh.

#### Configure TLS Origination

The egress gateway can then be configured to call `httpbin.org` over TLS.
We use a `DestinationRule` to capture the traffic policy and associate it with the host:

```shell
kubectl apply -f - <<EOF
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin.org-tls
  namespace: common-infrastructure
spec:
  host: httpbin.org
  trafficPolicy:
    tls:
      mode: SIMPLE
EOF
```

TLS origination is the mirror-opposite of TLS termination, and requires less configuration since with simple TLS, only the server identifies itself to the client.

### Test it

In one terminal, tail the egress gateway logs:

```shell
kubectl logs --follow -n common-infrastructure deployments/egress-gateway
```

From another terminal, send a request from the `curl` client to the external endpoint:

```shell
kubectl exec deploy/curl -- curl -s -v http://httpbin.org/get
```

Here is a sample log output (broken into multiple lines for legibility):

```console
[2025-06-30T02:48:20.634Z] "GET /get HTTP/1.1" 200 - via_upstream - "-" 0 255 1477 1477 "-" 
  "curl/8.14.1" "c29d1bce-1dcd-4615-a7e6-2e1caa35ce76" "httpbin.org" 
  "44.207.188.95:443" inbound-vip|80|http|httpbin.org 
  10.42.0.8:40324 240.240.0.3:80 10.42.0.9:35662 - default
```

### Add an Authorization Policy

The value of routing calls to external endpoints through an egress gateway is that it gives us a policy enforcement point (PEP).
We can configure an authorization policy at that egress gateway point to control _who_ is allowed to make _what kinds_ of calls to _which_ endpoints.
That is the essence of authorization.

TODO: walk through an example

### Analysis

The API we used to configure egress is familiar, the same API we use for ingress and, as you will see, mesh traffic.

The resources we used were:

The Kubernetes Gateway API's `Gateway` resource is used for provisioning the egress Gateway.  We use the same resource to configure Ingress Gateways and Waypoints.  The same API.

To control traffic management, we used Istio's `ServiceEntry` to define the target external endpoint, and `DestinationRule` to configure how (over simple TLS) we communicate with that destination.

Finally, security (authorization, to be precise) was configured using Istio's `AuthorizationPolicy` resource.

It should be noted that Istio also implicitly configures workload identity and mutual TLS-encrypted traffic within the mesh (no configuration required), as well as basic observability.
As we saw in the example, aspects of observability can be further configured with the `Telemetry` resource.

The same basic API is used to configure both ingress and east-west traffic.

## Configure & Control Mesh traffic

TODO

## Summary

- The Kubernetes Gateway API becomes a standard, not just for configuring ingress, but for all traffic flows.
- We provision ingress gateways, waypoints, and egress gateways the same way, using the `Gateway` resource.
    On-demand provisioning of gateways removes a major past pain point.
- We configure routing with the `HTTPRoute` resource.  `HTTPRoute` offers a powerful API to control routing, with support for many route matching rules and expressions.
    For fine-grained control over the mode of communication (protocols, encryption, etc..) we use for specific target workloads, we use a `DestinationRule`.
- Authorization is configured with the `AuthorizationPolicy` resource.
    When Layer 7 authorization is required, we configure a waypoint to proxy the services in question.