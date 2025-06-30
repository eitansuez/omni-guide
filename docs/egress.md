# Configure & Control Egress

Below, we walk through a general example for configuring egress, taken from [the documentation](https://ambientmesh.io/docs/traffic/mesh-egress/).

## Provision a gateway

Just like we did for ingress, we begin by provisioning a Gateway.

This time we enlist the help of the Istio CLI, specifically the command `istioctl waypoint apply`:

```shell
istioctl waypoint apply --enroll-namespace --name egress-gateway \
  --namespace common-infrastructure
```

The above command generates a Gateway resource and applies it to the indicated namespace.

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

### Create a ServiceEntry

The `ServiceEntry` resource is designed to allow Istio to be aware of services that reside outside the mesh.

Here is for example a `ServiceEntry` that communicates to Istio that mesh workloads may call the hostname `httpbin.org`.

```yaml
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
```

In this case, the IP address of the target workload is obtained through DNS resolution.

The convention is that clients call `httpbin.org` using the url `http://httpbin.org/` on port 80.
The ServiceEntry however, specifies a `targetPort` of 443, indicating that the outbound request to the actual service will be on a different port.

Since the `ServiceEntry` is defined in the `common-infrastructure` namespace, traffic from clients targeting the host `httpbin.org` will be routed to the egress gateway.
Communication from the client to the gateway will use mutual TLS by virtue of both ends of that communication being a part of the mesh.

### Configure TLS Origination

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

## Test it

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

## Add an Authorization Policy

Routing calls to external endpoints through an egress gateway effectively gives us a Policy Enforcement Point (PEP): we can configure an authorization policy at that egress gateway point to control _who_ is allowed to make _what kinds_ of calls to _which_ endpoints.
That is the essence of authorization.

TODO: walk through an example

## Analysis

The API we used to configure egress is familiar, the same API we use for ingress and, as you will see, mesh traffic.

The resources we used were:

- The Kubernetes Gateway API's `Gateway` resource is used for provisioning the egress Gateway.  We use the same resource to configure Ingress Gateways and Waypoints.  The same API.

- To control traffic management, we used Istio's `ServiceEntry` to define the target external endpoint, and `DestinationRule` to configure how (over simple TLS) we communicate with that destination.

- Finally, security (authorization, to be precise) was configured using Istio's `AuthorizationPolicy` resource.

It should be noted that Istio also implicitly configures workload identity and mutual TLS-encrypted traffic within the mesh (no configuration required), as well as basic observability.
As we saw in [setup](setup.md#configure-access-logging), aspects of observability can be further configured with the [`Telemetry`](https://istio.io/latest/docs/reference/config/telemetry/) resource.

The same basic API is used to configure both ingress and east-west traffic.