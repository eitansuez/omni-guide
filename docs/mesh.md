# Configure & Control Mesh traffic

Back in [setup](setup.md#deploy-a-set-of-microservices), when you deployed the `bookinfo` sample application, you were instructed to label the `bookinfo` namespace:

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
Let us work through an example.

## Traffic Management Example

A canonical example of traffic management in the mesh is to configure traffic splitting, which is the basis for performing canary deployments and A/B tests.

Note:  there may exist specific mesh features that were supported by Istio's original API that may still be considered experimental in the Gateway API.

In this example we focus on the `reviews` service, for which three versions happen to be deployed: v1, v2, and v3.

We begin by provisioning a waypoint.

## Provision a waypoint.

Traffic splitting, and the conditional routing of HTTP requests, perhaps based on specific headers in the request, are by definition a layer 7 concern, and so require that services be proxied by a layer 7 gateway.

In Istio ambient mode, we called these internal gateways "waypoints"; the term "micro gateway" is often used to refer to them.

As we already saw in the egress example, the Istio CLI provides a convenience command to provision a waypoint and optionally associate it to specific workloads.

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

## Setup

In ambient mode, we are no longer required to define "subsets" - specific versions of a microservice - with a `DestinationRule`.
We can simply use a Kubernetes `Service` definition with the right combination of selectors.

Apply the following resource (which is bundled with the `bookinfo` sample app), which defines these "sub" services for `bookinfo`, among them `reviews-v1`, `reviews-v2`, and `reviews-v3`:

```shell
kubectl apply -n bookinfo \
  -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo-versions.yaml
```

Now that we have specific services defined for each version of the `reviews` service, we can define HTTPRoutes that reference them.

## Configure the waypoint

Configure a traffic split whereby 50% of requests are routed to the service `reviews-v1` and 50% to `reviews-v2` (implying none to `reviews-v3`).

The following `HTTPRoute` captures the configuration:

```yaml
--8<-- "reviews-route.yaml"
```

Above, the difference in configuration from the ingress routing example is the `parentRef` being of type Service -- previously it was a reference to a Gateway.
By associating the route with a Service, we are basically configuring routing rules for any requests that target that service.

Apply the route.

Inspect the routes in the `bookinfo` namespace:

```shell
kubectl get httproute -n bookinfo
```

The output shows that we now have two routes, one applied at ingress, bound to the gateway, and the other the route that configures internal traffic to the `reviews` service:

```console
NAME             HOSTNAMES                  AGE
bookinfo-route   ["bookinfo.example.com"]   42m
reviews-route                               4s
```

## Test it

```shell
for i in {1..10}; do
kubectl exec deploy/curl -- curl -s reviews.bookinfo:9080/reviews/123 | jq | grep podname
done
```

Note in the output that responses are coming exclusively from instances of `reviews-v1` and `reviews-v2` services, with approximately a 50% split:

```console
  "podname": "reviews-v2-5c757d5846-b57dx",
  "podname": "reviews-v1-849f9bc5d6-qs8nx",
  "podname": "reviews-v1-849f9bc5d6-qs8nx",
  "podname": "reviews-v2-5c757d5846-b57dx",
  "podname": "reviews-v1-849f9bc5d6-qs8nx",
  "podname": "reviews-v1-849f9bc5d6-qs8nx",
  "podname": "reviews-v2-5c757d5846-b57dx",
  "podname": "reviews-v2-5c757d5846-b57dx",
  "podname": "reviews-v1-849f9bc5d6-qs8nx",
  "podname": "reviews-v1-849f9bc5d6-qs8nx",
```

We can proceed to configure other aspects of the mesh, such as adding an `AuthorizationPolicy` for calls to the `reviews` service, to control what workloads are permitted to interact with it.  The process would be similar to examples we've already seen for ingress and egress.

## Analysis

In this mesh example, we applied the same API, but to control internal traffic.

The overall procedure was very similar to ingress and egress:

- Provision the gateway or waypoint.
- Associate the proxy with specific services (not applicable for the ingress use case).
- Define traffic policies (routing rules and others) such as HTTPRoutes and possibly DestinationRule's.
- Apply security policies such as AuthorizationPolicy resources.
