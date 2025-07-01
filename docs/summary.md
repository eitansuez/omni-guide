# Summary

In this guide, we have seen how a single API can be used in Istio ambient mesh to configure ingress, egress, and mesh traffic.

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

- makes it easy to [integrate third-party API gateways](https://ambientmesh.io/docs/traffic/third-party-gateways/) with mesh workloads.
- supports the notion of [pluggable waypoints](https://kgateway.dev/blog/extend-istio-ambient-kgateway-waypoint/) - bringing your own layer 7 proxy to configure aspects of the mesh, opening the door for additional capabilities such as rate limiting and request transformations.