# Summary

- The Kubernetes Gateway API becomes a standard, not just for configuring ingress, but for all traffic flows.
- We provision ingress gateways, waypoints, and egress gateways the same way, using the `Gateway` resource.
    On-demand provisioning of gateways removes a major past pain point.
- We configure routing with the `HTTPRoute` resource.  `HTTPRoute` offers a powerful API to control routing, with support for many route matching rules and expressions.
    For fine-grained control over the mode of communication (protocols, encryption, etc..) we use for specific target workloads, we use a `DestinationRule`.
- Authorization is configured with the `AuthorizationPolicy` resource.
    When Layer 7 authorization is required, we configure a waypoint to proxy the services in question.