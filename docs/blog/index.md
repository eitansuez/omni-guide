# The power of a single API to secure, observe, and control traffic in all directions

## Introduction

In the latest releases of Istio, with support for Ambient mode becoming Generally Available (GA), and with its support for the Kubernetes Gateway API, we are finally at a point where platform engineers can provision, configure and control network traffic in uniform way, and without the overhead of sidecars.

It is important to stress that the above term "network traffic" implies traffic in all directions: at ingress, at egress, as well as internal "mesh" traffic, also known as "east-west" traffic.
This vision of a single, uniform API to control traffic in all directions is something we at Solo.io have dubbed "Omni" (see [What is Omni?](https://www.solo.io/resources/video/what-is-omni-part-1-of-2)).

To get a better sense for this process, we will explore how to configure ingress, egress, and mesh traffic using this uniform API.

## Context

The context for this exploration is a system running Istio in ambient mode.

This entails a Kubernetes cluster with the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) CRDs installed, and [Istio installed with the ambient profile](https://istio.io/latest/docs/ambient/install/).

On that platform are deployed sets of microservices that represent applications and APIs consumed by the enterprise and its customers.

Istio's canonical example is the [`bookinfo` sample application](https://istio.io/latest/docs/examples/bookinfo/), which consists of the microservices `productpage`, `details`, `reviews` and `ratings`.

Istio ambient mesh represents an evolution of the previous sidecar model, where platform concerns are no longer in any way coupled to your workloads:  no sidecars are required, implying that workloads don't need to be restarted to inject them, neither are restarts required when you upgrade Istio.

The typical way in which we add our workloads to the mesh is by labeling their namespaces:

```shell
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

Ultimately, applications and APIs need to be exposed to the users who consume them.
Let us begin by discussing securing and controlling ingress traffic.

## Ingress

The main task when configuring ingress is to expose an application or API to the internet through an ingress gateway.

The process involves:

- configuring and provisioning gateways, 
- configuring traffic flows, and 
- overlaying authorization policies.

### The Gateway API

How we configure ingress on Kubernetes has evolved over time.
Originally Kubernetes provided the [Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/).
Projects that implemented ingress in Kubernetes often came up with, and offered their own alternative CRDs.
One example is Istio's original API and specifically the Gateway and VirtualService resources.
Most recently a new effort was launched to produce and updated standard, the Kubernetes Gateway API to address the shortcomings of the previous model.

Today the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) is stable and, at the time of writing, at version 1.3.0.

Istio ambient mesh aligns with, and depends on this new standard.
The older Istio resources continue to be supported, but with Istio ambient, the Gateway API becomes the primary interface.

### Configuring the Gateway

Gateways represent a point of entry into a cluster.
Gateways should typically be configured to:

- Accept HTTPS traffic:  communications with a client should be encrypted.
- Redirect HTTP traffic to the HTTPS scheme:  plain text traffic should not be accepted, but politely redirected.
- Terminate TLS:  the gateway is often the point at which we configure the TLS certificate used in the communications handshake with clients.  Traffic from the Gateway to an "upstream" workload should be considered mesh-traffic and employ mutual TLS, something that Istio takes care of for us.

We also have the option to deploy either dedicated gateways for specific applications, or a shared gateway that routes ingress traffic on behalf of mulitple APIs and applications.

Here is a representative configuration for Istio's `bookinfo` sample application:

```yaml
--8<-- "gateway.yaml"
```

Above, we:

- Name the gateway `shared-gateway`, and place it in the `common-infrastructure` namespace.
- Associate the ingress gateway with Istio by specifying the `istio` gatewayClassName.
- Expose a listener on port 80 that we intend to configure to redirect traffic to port 443.
- Expose a listener on port 443, bound to the [ficticious] hostname `bookinfo.example.com`, configured to terminate TLS.
- Configure the gateway to allow the binding of routes that reside in namespaces labeled `self-serve-ingress` with value `true` (an adhoc convention).

Part of this effort involves the generation of the TLS certificate for the hostname and its storage into a kubernetes `Secret` by the name of `bookinfo-cert`.  This can be performed using many different tools and methods.

### Provisioning the Gateway

Historically the deployment of the Gateway was an installation-time concern, and its configuration separate.
This approach was problematic and turned out to be a pain point for platform engineers.
Gateways are a part of the data plane, and require more flexibility to provision when "building out" the traffic flows we need to support.
The Gateway API changes the approach and allows for on-demand provisioning of Gateways.

Applying the `Gateway` resource to a cluster triggers a number of actions:

1. A gateway is provisioned:  Istio will provision an Envoy proxy for the gateway.  You will see a deployment in the target namespace  (which begets a ReplicaSet and in turn a Pod).
2. Istio will program the proxy according to the `Gateway` resource's specification.
3. A service of type LoadBalancer is created.  For a Kubernetes cluster deployed in the cloud, this implies the automatic creation of an L4-type load balancer with a public static IP address, configured to route requests to the Envoy proxy workloads (a DNS record is then configured for hosts, pointing to that public IP address).

At this point the Gateway can receive HTTP requests from the internet, but will return a 404 "Not Found", until we configure how we wish the proxy to route those requests.

### Controlling routing

The Gateway API offers protocol-specific Route resources such as HTTPRoute and GRPCRoute.

To gain familiarity with the API, let us study the HTTPRoute resource bundled with Istio's `bookinfo` sample application to expose the `productpage` service's endpoints:

```yaml
--8<-- "bookinfo-route.yaml"
```

It is worth calling out how the route binds to the Gateway through the `parentRefs` field, and that we bind specifically to the port 443 listener:  when a request arrives over HTTPS, the routing rules will match.  We expose various endpoints including the `productpage` web UI, access to static resources, the `/login` and `/logout` endpoints, and the `productpage` API endpoints (prefixed by `/api/v1/products`).  The `backendRef` directs matching requests to the service running inside the cluster.

The Gateway API offers many more controls over how requests get routed, including traffic splitting, request mirroring, support for timeouts and retries.  Istio has had its own resources for traffic management, namely the [`VirtualService`](https://istio.io/latest/docs/reference/config/networking/virtual-service/) resource, and today fully supports the Gateway API as the de facto standard.

The same `HTTPRoute` (and other Route-type resources) is also used for configuring east-west traffic, as we will see in the "mesh traffic" section.

### Add an AuthorizationPolicy

Istio allows us to overlay a security configuration as well.

For example, we can define an AuthorizationPolicy to allow only the Ingress gateway to call the `productpage` service.

```yaml
--8<-- "productpage-authz.yaml"
```

Above, note the [SPIFFE](https://spiffe.io/docs/latest/spiffe-about/overview/) identity of the Gateway `shared-gateway` is a function of its namespace and associated service account name.

This authorization policy is not strictly an ingress-specific concern, as it applies to any requests targeting the `productpage` service.

### Analysis

At this point our configuration should be functioning.
We have a Gateway provisioned, configured for HTTPS, with a route resource bound to the gateway that governs the routing of requests to an upstream service.

Requests to the configured hostname will flow first to the cloud load balancer, then to the Envoy proxy that is our ingress gateway, and from there to our backing workloads.

Furthermore, the AuthorizationPolicy resource ensures that no other workloads are allowed to make requests to the `productpage` service.

Here is a general outline of the process we followed to configure ingress:

- Provision and configure the gateway with the `Gateway` resource
- Configure routing with the `HTTPRoute` (or a different protocol-specific route) resource
- Configure security policy

Let us next look at how we might configure and control egress traffic, and see how the process compares to what we just outlined for ingress.

## Egress

Let us walk through a general example for configuring egress, taken from [the documentation](https://ambientmesh.io/docs/traffic/mesh-egress/).

### Provision a gateway

Just like we did for ingress, we begin by provisioning a Gateway.

This time we enlist the help of the Istio CLI, specifically the command `istioctl waypoint apply`:

```shell
istioctl waypoint apply --enroll-namespace --name egress-gateway \
  --namespace common-infrastructure
```

The above command generates a Gateway resource and applies it to the indicated namespace.

The GatewayClass for the egress gateway will be `istio-waypoint` (whereas for ingress it was just `istio`).

The other subtle but important point in the above `istioctl` command is the `--enroll-namespace` flag:  any services defined in that namespace will be proxied by this egress gateway.

### Create a ServiceEntry

The `ServiceEntry` resource is designed to allow Istio to be aware of services that reside outside the mesh.

Here for example is a `ServiceEntry` that represents the external service running on the internet at `httpbin.org`:

```yaml
--8<-- "service-entry.yaml"
```

Note the `resolution: DNS` specification, which tells Istio to resolve the IP address of the target service with DNS resolution.

The convention is that clients call `httpbin.org` using the url `http://httpbin.org/` on port 80.
The ServiceEntry however, specifies a `targetPort` of 443, indicating that the outbound request to the actual service will be on a different port.

Since the `ServiceEntry` is defined in the same namespace as the egress gateway, traffic from clients targeting the host `httpbin.org` will be routed to the egress gateway.

Communication from a client to the gateway will use mutual TLS by virtue of both ends of that communication being a part of the mesh.

The egress gateway can then be configured to call `httpbin.org` over TLS.

### Configuring TLS Origination

We use a `DestinationRule` to capture the desired traffic policy, and associate it with the host:

```yaml
--8<-- "tls-origination.yaml"
```

TLS origination is the mirror-opposite of TLS termination, and requires less configuration since with simple TLS, only the server identifies itself to the client.

### Add an Authorization Policy

Routing calls to external endpoints through an egress gateway effectively gives us a Policy Enforcement Point (PEP): we can configure an authorization policy at that egress gateway point to control _who_ is allowed to make _what kinds_ of calls to _which_ endpoints.
That is the essence of authorization.

Study the following example, an AuthorizationPolicy that allows only HTTP GET-type operations to the external `httpbin` service's `/get` endpoint and nothing else:

```yaml
--8<-- "httpbin-authz.yaml"
```

Applying the above policy to the Kubernetes cluster is all it takes to enforce it.

### Analysis

At this point, egress to `httpbin.org` should be functioning:

- A workload running in the mesh can send a request to that url, and Istio will route the request through the egress gateway which will then call `httpbin.org` over HTTPS.
- If clients attempt to call unauthorized endpoints or use an unauthorized HTTP method, the request will be denied.

The API we used to configure egress is familiar, the same API we use for ingress and, as you will see, mesh traffic.

The resources we used were:

- The Kubernetes Gateway API's `Gateway` resource is used for provisioning the egress Gateway.  We use the same resource to configure Ingress Gateways and Waypoints.  The same API.

- To control traffic management, we used Istio's `ServiceEntry` to define the target external endpoint, and `DestinationRule` to configure how (over simple TLS) we communicate with that destination.

- Security (authorization, to be precise) was configured using Istio's `AuthorizationPolicy` resource.

## Mesh

Back in [_Context_](#context), we noted Istio ambient's convention for adding workloads to the mesh:

```shell
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

That simple instruction was all that was required to make the workloads _a part of the mesh_.
Unlike sidecar mode, with ambient mode workloads do not require a restart, because no sidecars need to be injected.
The mesh concerns reside in a separate layer, and become a part of the underlying platform.

It's important to understand and to highlight what it means exactly for a workload to be "a part of the mesh."
What happens behind the scenes?

The following takes place implicitly, without requiring any configuration:

- Workloads are assigned cryptographic identities by the control plane (ztunnel maintains them on behalf of your workloads).
- The Istio CNI plugin reroutes the network communication in and out of each pod through the ztunnel layer 4 node proxies.
- The ztunnel proxies leverage those workload identities to encrypt all network communication between workloads using mutual TLS.
- The ztunnel proxies automatically produce and expose layer 4 telemetry: TCP bytes sent and received, TCP connections opened and closed.

The mesh doesn't require any additional configuration, but it definitely has more to offer.
We can:

- Layer on L4 authorization policies, restricting network communications based on workload identity
- Add L7 traffic management policies, leveraging higher level information for making routing decisions
- Add L7 authorization policies, again leveraging higher level information for making security-related decisions
- Add L7 telemetry

All of those layer 7 capabilities rely on layer 7 proxies, which ambient mode allows us to provision on demand.
Let us look at an example.

### Example

A canonical example of traffic management in the mesh is to configure traffic splitting, which is the basis for performing canary deployments and A/B tests.

In this example we focus on the `reviews` service, for which three versions happen to be deployed: v1, v2, and v3.

We begin by provisioning a waypoint (a procedure that, by now, begins to feel familiar).

#### Provision a waypoint

Traffic splitting, and the conditional routing of HTTP requests, perhaps based on specific headers in the request, are by definition a layer 7 concern, and so require that services be proxied by a layer 7 gateway.

In Istio ambient mode, we called these internal gateways "waypoints"; the term "micro gateway" is often used to refer to them.

As we saw in the egress example, the Istio CLI provides a convenience command to provision a waypoint and optionally associate it to specific workloads.

In this example, we provision the waypoint `bookinfo-proxy` and associate it to all workloads that reside in the `bookinfo` namespace:

```shell
istioctl waypoint apply --enroll-namespace --name bookinfo-proxy \
  --namespace bookinfo
```

The implication is that any request to any of the `bookinfo` services will be proxied by this newly-created waypoint.

What the command actually does to "enroll" the namespace is to label it with the conventional label `istio.io/use-waypoint`, as indicated by the output of the above command:

```console
✅ waypoint bookinfo/bookinfo-proxy applied
✅ namespace bookinfo labeled with "istio.io/use-waypoint: bookinfo-proxy"
```

#### Setup

In ambient mode, we are no longer required to define "subsets" - specific versions of a microservice - with a `DestinationRule`.
We can simply use a Kubernetes `Service` definition with the right combination of selectors.

With specific services defined for each version of the `reviews` service, we can define HTTPRoutes that reference them.

#### Configure the waypoint

Configure a traffic split whereby 50% of requests are routed to the service `reviews-v1` and 50% to `reviews-v2` (implying none to `reviews-v3`).

The following `HTTPRoute` captures the configuration:

```yaml
--8<-- "reviews-route.yaml"
```

Above, the difference in configuration from the ingress routing example is the `parentRef` being of type Service -- previously it was a reference to a Gateway.
By associating the route with a Service, we are basically configuring routing rules for any requests that target that service.

Applying the above HTTPRoute resource triggers the programming of the waypoint to route requests according to the specification provided: 50% to v1 and 50% to v2.

We can proceed to configure other aspects of the mesh, such as adding an `AuthorizationPolicy` for calls to the `reviews` service, to control what workloads are permitted to interact with it.
The process would be similar to examples that we already saw for ingress and egress.

### Analysis

In this mesh example, we applied the same API, but to control internal traffic.

The overall procedure was very similar to ingress and egress:

- Provision the gateway or waypoint.
- Associate the proxy with specific services (not applicable for the ingress use case).
- Define traffic policies (routing rules and others) such as HTTPRoutes and possibly DestinationRule's.
- Apply security policies such as AuthorizationPolicy resources.

## Summary

We have seen how a single API can be used in Istio ambient mesh to configure ingress, egress, and mesh traffic.

We configured a mesh to control ingress traffic, egress traffic, as well as mesh traffic.

We can draw the following observations:

- The Kubernetes Gateway API becomes a standard, not just for configuring ingress, but for all traffic flows.
- We provision ingress gateways, waypoints, and egress gateways in similar ways, using the `Gateway` resource.
    On-demand provisioning of gateways removes a major past pain point.
- We configure routing for both ingress and mesh traffic with the same `HTTPRoute` resource.  `HTTPRoute` offers a powerful API to control routing, with support for many route matching rules and expressions.
    For fine-grained control over the mode of communication (protocols, encryption, etc..) we use for specific target workloads, we use a `DestinationRule`.
- Authorization is configured with the `AuthorizationPolicy` resource.
    When Layer 7 authorization is required, we configure a waypoint to proxy the services in question.

There is more to the story.  Istio ambient mesh:

- Makes it easy to [integrate third-party API gateways](https://ambientmesh.io/docs/traffic/third-party-gateways/) with mesh workloads.
- Supports the notion of [pluggable waypoints](https://kgateway.dev/blog/extend-istio-ambient-kgateway-waypoint/) - bringing your own layer 7 proxy to configure aspects of the mesh, opening the door for additional capabilities such as rate limiting and request transformations.