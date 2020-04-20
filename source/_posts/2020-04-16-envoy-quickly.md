---
title: "Observability by deploying Envoy without a control plane"
date: 2020-04-19 15:30:00
tags:
  - envoy
  - microservices
  - kubernetes
  - observability
---

{% asset_img comic.png REEEEEE %}

Epilogue:

"I was so sure that a service mesh would do the trick" Bob thought. "Would it be possible to re-purpose Envoy to just be a forward proxy? Let Kubernetes do service discovery so we don't have to rely on a control plane about new upstreams anymore.

"A transparent proxy that auto-magically gathers metrics seems to be a good idea." Bob trailed off.

- - -

Looks like Bob is onto something there and lucky for us, [Envoy now supports HTTP dynamic proxy!](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_proxy) Let's try to validate whether Bob was on the right track!

The complete and usable code for this post can be found [here](https://github.com/teh-username/envoy-quickly).

## Here Be Dragons

_Note: Envoy's forward proxy implementation is still considered alpha and not production ready._

As with any solution, this idea is not a silver bullet. It can only be applied to particular development environments (painstakingly described in the introductory comic). If the comic hits home for you, then you might be able to get something out of this. If not, it's still an interesting approach I think!

## Passthrough Approach

Just so we are on the same page, our main goal is:

> Given a pre-existing microservice-based backend deployed in Kubernetes, setup a metrics dashboard for all services as quickly as possible (considering all the constraints set by the intro comic).

Our strategy involves configuring Envoy as a forward proxy. Injecting this in front of our services shouldn't change any behavior (it's just a passthrough) and we get the benefit of metrics collection in the background. Win-win!

One (positive?) side-effect of this approach is that it effectively removes the need for a control plane (as we push host discovery back to Kubernetes). There's no need for the sidecars to poll for config updates anymore, further reducing the complexity of the solution.

With that settled, let's see it in action!

## Deployment Strategy

To illustrate how the idea works, we'll be using [Bouyant's](https://buoyant.io/) sample app, [emojivoto](https://github.com/BuoyantIO/emojivoto) (which is also used by [Linkerd](https://linkerd.io/) as its demo app) as our "pre-existing" application.

Once bootstrapped with proxy sidecars, our application will look like:

```
+--------------+     +-------------+     +-------------+     +--------------+
| +----------+ |     |   +-----+   |     |  +-------+  |     |  +--------+  |
| | vote-bot | |     |   | web |   |     |  | emoji |  |     |  | voting |  |
| +----------+ |     |   +-----+   |     |  +-------+  |     |  +--------+  |
|      ^       |     |      ^      |     |      ^      |     |       ^      |
|      |       |     |      |      |     |      |      |     |       |      |
|      v       |     |      v      |     |      v      |     |       v      |
|   +-----+    |     |   +-----+   |     |   +-----+   |     |    +-----+   |
|   |proxy|    |     |   |proxy|   |     |   |proxy|   |     |    |proxy|   |
|   +-----+    |     |   +-----+   |     |   +-----+   |     |    +-----+   |
+--------------+     +-------------+     +-------------+     +--------------+
       ^                 ^  ^  ^                ^                    ^
       |       REST      |  |  |      gRPC      |                    |
       +-----------------+  |  +----------------+                    |
                            |                                        |
                            +----------------------------------------+
                                               gRPC
```

To achieve this, we have to edit our service manifests in 2 locations:

1) `initContainers` - We run a container that updates the iptables of the pod so that all ingress and egress traffic is redirected to Envoy automatically. This removes the need to update the services to be aware of Envoy.

2) `containers` - We add Envoy as another container in the pod spec.

To know more about these changes, check out my other [repository](https://github.com/teh-username/service-mesh-the-hard-way) where we try to build our own service mesh from scratch!

## Envoy Configuration

We'll be running with the following configuration:

#### Ingress
* Cluster
  * `grpc_local_service` pointed at `127.0.0.1:8080`, uses HTTP/2 (GRPC)
  * `rest_local_service` pointed at `127.0.0.1:8080`, defaults to HTTP/1.x

* Listener
  * `0.0.0.0:9211` as ingress with the following routes:
    * `grpc_route` (if "application/grpc") routes to `grpc_local_service`
    * `rest_route` (default route) routes to `rest_local_service`

#### Egress
* Cluster
  * `grpc_proxy_cluster` as a dynamic forward proxy, uses HTTP/2 (GRPC)
  * `rest_proxy_cluster` as a dynamic forward proxy, defaults to HTTP/1.x

* Listener
  * `127.0.0.1:9001` as egress with the following routes:
    * `grpc_route` (if "application/grpc") routes to `grpc_proxy_cluster`
    * `rest_route` (default route) routes to `rest_proxy_cluster`

You'll notice that we had to separate the upstreams that communicate over HTTP/1.x and HTTP/2. This is because if we hint Envoy that a particular upstream supports HTTP/2, [Envoy will talk to that upstream over HTTP/2 only](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster.proto#envoy-api-field-cluster-http2-protocol-options).

## Results

<div style='position:relative; padding-bottom:54%'><iframe src='https://gfycat.com/ifr/greedyseverebluet' frameborder='0' scrolling='no' width='100%' height='100%' style='position:absolute;top:0;left:0;' allowfullscreen></iframe></div>

You can find more info about Envoy's statistics [here](https://www.envoyproxy.io/docs/envoy/latest/operations/stats_overview) and [here](https://docs.datadoghq.com/integrations/envoy/#data-collected).

## Conclusion

Now that we've gone through the entire implementation, here are some closing points to keep in mind should you consider adopting this solution:

* This is intended as a quick, interim solution while you are off building a much better one.
* There's a huge assumption that all of your services serve traffic on the same port.
* Envoy's configuration complexity is directly proportional to your architecture complexity.
* The configuration has to be sanity checked every time there's a release. This can be automated though.
* We're missing out on other big features of Envoy that service meshes take advantage of.

Happy monitoring!
