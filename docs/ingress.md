# Configure & Control Ingress

As you review the steps outlined below, observe the general process we follow to provision gateways, configure traffic flows, and overlay authorization policies.

The main task is to expose `bookinfo`'s web UI and API (served by the `productpage` service) to the internet through an ingress gateway.

## Provision and configure the Gateway

Before we begin, we need to make some design decisions that pertain to our gateway:

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

Let us break down the above configuration; we:

- Name the gateway `shared-gateway`, and place it in the `common-infrastructure` namespace.
- Associate the ingress gateway with Istio by specifying the `istio` gatewayClassName.
- Expose a listener on port 80 that we intend to configure to redirect traffic to port 443.
- Expose a listener on port 443, bound to the [ficticious] hostname `bookinfo.example.com`, configured to terminate TLS.
- Configure the gateway to allow the binding of routes that reside in namespaces labeled `self-serve-ingress` with value `true` (an adhoc convention).

Apply that label to the `bookinfo` namespace now, since we know we will need to define a route there for directing requests to the `productpage` service:

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

## Controlling routing

The HTTPRoute resource to expose the `productpage` service's endpoints is bundled with Istio's `bookinfo` sample application.  Here it is, adjusted to this specific context:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-route
  namespace: bookinfo
spec:
  hostnames:
  - bookinfo.example.com
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

The Gateway API offers many more controls over how requests get routed, including traffic splitting, request mirroring, support for timeouts and retries.  Istio has had its own resources for traffic management, namely the [`VirtualService`](https://istio.io/latest/docs/reference/config/networking/virtual-service/) resource, and today fully supports the Gateway API as the defacto standard.

The same `HTTPRoute` (and other Route-type resources) is also used for configuring east-west traffic, as we will see shortly.

## Add an AuthorizationPolicy

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

## Test ingress

We can obtain the external IP address of the ingress gateway either by inspecting the Service resource `status` section.

```shell
kubectl get svc -n common-infrastructure shared-gateway-istio
```

```console
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                                      AGE
shared-gateway-istio   LoadBalancer   10.43.94.151   192.168.97.2   15021:32006/TCP,80:31344/TCP,443:32559/TCP   175m
```

Or as an environment variable `GW_IP`:

```shell
export GW_IP=$(kubectl get svc -n common-infrastructure shared-gateway-istio -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Alternatively, we can inspect the Gateway resource's `status` section:

```shell
kubectl get gtw -n common-infrastructure shared-gateway
```

```console
NAME             CLASS            ADDRESS        PROGRAMMED   AGE
shared-gateway   istio            192.168.97.2   True         175m
```

 Capture the IP address as an environment variable:

```shell
export GW_IP=$(kubectl get gtw -n common-infrastructure shared-gateway -o jsonpath='{.status.addresses[0].value}')
```

With DNS to resolve to that address, we should be able to simply send a `curl` request.

Alternatively (without DNS configured), we can instruct `curl` to resolve the hostname to that IP address as follows:

```shell
curl -s --insecure https://bookinfo.example.com/productpage --resolve bookinfo.example.com:443:$GW_IP | grep title
```

The response is an HTML page, and the above command just looks for the page title (to limit the verboseness of the output):

```console
<title>Simple Bookstore App</title>
```

This is evidence that ingress is configured and functioning.

## Test authorization

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

## Analysis for Ingress

Here is a general outline of the process we followed to configure ingress:

- Provision and configure the gateway with the `Gateway` resource
- Configure routing with the `HTTPRoute` (or a different protocol-specific route) resource
- Configure security policy

Let us next look at how we might configure and control egress traffic, and see how the process resembles what we just did for ingress.