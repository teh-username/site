---
title: Sanity checking Istio's mTLS on permissive mode
date: 2019-10-11 22:09:52
tags:
  - kubernetes
  - mtls
  - istio
  - envoy
---

One of the activities in Istio's [Mutual TLS Deep-Dive](https://istio.io/docs/tasks/security/mutual-tls/) task is the verification of the mTLS connection using curl requests inside the Envoy sidecar (instead of having Envoy do it for us). Everything works just fine but things start to get strange when we try the same curl requests with the server set to permissive mode (accept both HTTP and mTLS connection) as shown below:

```
HOST:PORT                                  STATUS     SERVER        CLIENT     AUTHN POLICY                DESTINATION RULE
httpbin.mtls-pg.svc.cluster.local:8000     OK         HTTP/mTLS     mTLS       httpbin.mtls-pg/mtls-pg     httpbin.mtls-pg/mtls-pg
```

> The output above was made possible by [`istioctl authn tls-check`](https://istio.io/docs/reference/commands/istioctl/#istioctl-authn-tls-check). See [aside](#Aside-Checking-mTLS-mappings) for more information.

CRDs applied are (see [aside](#Aside-Policies-and-Destination-Rules) for more information):

```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "httpbin.mtls-pg"
  namespace: "mtls-pg"
spec:
  targets:
  - name: httpbin
  peers:
  - mtls:
      mode: PERMISSIVE
---
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin.mtls-pg"
  namespace: "mtls-pg"
spec:
  host: httpbin.mtls-pg.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

Interestingly enough, when I forced the curl request to use mTLS I got:

```
000
command terminated with exit code 35
```

however the plain HTTP ones work. I reverted back to strict mTLS mode just to make sure I didn't bork my request and it worked. Does this mean permissive mode is equivalent to only accepting HTTP requests?

### Oh-so-magical Envoy

I played around with Istio a bit and see if I misconfigured something but to no avail so I hopped on the internet and started searching. I then came across this [issue](https://discuss.istio.io/t/policy-in-permissive-mode-mode-mutual-in-dr-https-mtls-doesnt-work/2228) which kind of resembles my problem but instead of using `ISTIO_MUTUAL` as the mode for the destination rule, they were using `MUTUAL` and a custom CA to boot. I quickly skimmed the thread and they started mentioning about potential problems with how Istio builds Envoy's configuration. So I took a peek inside the sidecar's configuration and sure enough, there lies the explanation to the problem I'm having (spoiler alert: mTLS works in permissive mode).

The following JSON file shows the server's sidecar listener configuration when set to use strict mTLS:

```json
{
    "name": "10.42.1.17_80",
    "address": {
        "socketAddress": {
            "address": "10.42.1.17",
            "portValue": 80
        }
    },
    "filterChains": [
        {
            "tlsContext": {
                ...snip...
            },
            "filters": [
                {
                    "name": "envoy.http_connection_manager",
                    ...snip...
                }
            ]
        }
    ],
    ...snip...
}
```

Now, this is what the configuration looks like when set to use permissive mode:

```json
{
    "name": "10.42.1.17_80",
    "address": {
        "socketAddress": {
            "address": "10.42.1.17",
            "portValue": 80
        }
    },
    "filterChains": [
        {
            "filterChainMatch": {
                "applicationProtocols": ["istio"]
            },
            "tlsContext": {
                ...snip...
            },
            "filters": [
                {
                    "name": "envoy.http_connection_manager",
                    ...snip...
                }
            ]
        },
        {
            "filterChainMatch": {},
            "filters": [
                {
                    "name": "envoy.http_connection_manager",
                    ...snip...
                }
            ]
        }
    ],
    ...snip...
}
```

Summarizing the listeners gives us:

* On strict mode:
    * Listener - mTLS support

* On permissive mode:
    * Listener 1 - mTLS support, application protocol should be `istio`
    * Listener 2 - HTTP support

So when we try our mTLS request, we get funneled to the plain HTTP listener (that does not accept mTLS) since our target listener is expecting `istio` as the application protocol (which we lack), causing the request to fail.

Our curl request works however in strict mTLS mode since Envoy only expects an mTLS connection but doesn't bother looking for `istio` as the application protocol.

Our observations are further confirmed when we take a look at the cluster configuration of the client service:

```json
{
    "name": "outbound|8000||httpbin.mtls-pg.svc.cluster.local",
    "type": "EDS",
    ...snip...
    "tlsContext": {
        "commonTlsContext": {
            ...snip...
            "alpnProtocols": [
                "istio"
            ]
        },
        ...snip...
    },
    ...snip...
}
```

Envoy sets the protocol to `istio` via the `alpnProtocols` configuration setting when the client is set to use mTLS.

### Actual proof of mTLS over permissive mode

Everything that we've gone through so far doesn't prove that Istio indeed uses mTLS in permissive mode, it only explains why it fails. Luckily, I was playing around with integrating [Open Policy Agent](https://www.openpolicyagent.org/) with Istio (tackled on a future entry) which gives our smoking gun.

Sending the request:

```
kubectl exec sleep-69c766786-662j7 -c sleep -n mtls-pg -- curl http://httpbin:8000/headers -o /dev/null -s -w '%{http_code}\n'
```

yielded an OPA input of:

```
{
    "decision_id": "5fa44190-d1c3-4032-ab01-42e414df0af4",
    "input": {
        "attributes": {
            "destination": {
                "address": {
                    "Address": {
                        "SocketAddress": {
                            "PortSpecifier": {
                                "PortValue": 80
                            },
                            "address": "10.42.1.17"
                        }
                    }
                },
                "principal": "spiffe://cluster.local/ns/mtls-pg/sa/default"
            },
            "request": {
                "http": {
                    "headers": {
                        ":authority": "httpbin:8000",
                        ":method": "GET",
                        ":path": "/headers",
                        "accept": "*/*",
                        "content-length": "0",
                        "user-agent": "curl/7.64.0",
                        "x-b3-sampled": "0",
                        "x-b3-spanid": "606d999688275b28",
                        "x-b3-traceid": "95fad1fde15e1130606d999688275b28",
                        "x-forwarded-client-cert": "By=spiffe://cluster.local/ns/mtls-pg/sa/default;Hash=0fd8a69e4977ba156e25cea9aad41bf56837da968c189c626d3d02c6be6e54e9;Subject=\"\";URI=spiffe://cluster.local/ns/mtls-pg/sa/sleep",
                        "x-forwarded-proto": "http",
                        "x-request-id": "77a17ed2-a6dd-4941-8d8f-470a7fbef83d"
                    },
                    "host": "httpbin:8000",
                    "id": "11404563429867098727",
                    "method": "GET",
                    "path": "/headers",
                    "protocol": "HTTP/1.1"
                }
            },
            "source": {
                "address": {
                    "Address": {
                        "SocketAddress": {
                            "PortSpecifier": {
                                "PortValue": 45090
                            },
                            "address": "10.42.0.49"
                        }
                    }
                },
                "principal": "spiffe://cluster.local/ns/mtls-pg/sa/sleep"
            }
        },
        "parsed_body": {},
        "parsed_path": [
            "headers"
        ],
        "parsed_query": {}
    },
    ...snip...
}
```

This is enough proof that Istio uses mTLS on permissive mode as well. Case closed!

### [Aside] Checking mTLS mappings

Istio comes with a command-line utility tool to help users debug/diagnose their mesh called [`istioctl`](https://istio.io/docs/reference/commands/istioctl/). One of the commands available to `istioctl` is [`authn tls-check`](https://istio.io/docs/reference/commands/istioctl/#istioctl-authn-tls-check) which basically checks what kind of connection a particular service can receive (as a server) and send (as a client).

For example, running:

`istio-1.3.2/bin/istioctl authn tls-check sleep-69c766786-662j7.mtls-pg`

which can be interpreted as "check the mTLS setting for the pod named `sleep-69c766786-662j7` in the `mtls-pg` namespace concerning all of the pods in the mesh" yields:

```
HOST:PORT                                                       STATUS           SERVER        CLIENT          AUTHN POLICY        DESTINATION RULE
...snip...

sleep.default.svc.cluster.local:80                              OK               HTTP/mTLS     HTTP            default/            -

...snip...
```

Focusing on the `SERVER` and `CLIENT` columns, we explain the output as follows:

* `sleep-69c766786-662j7.mtls-pg` sees the service `sleep.default` as a server that can accept either plaintext (HTTP) or mTLS connections (permissive mode). We can tweak this using [Policies](https://istio.io/docs/reference/config/istio.authentication.v1alpha1/) which will show up in the `AUTHN POLICY` column.

* When `sleep-69c766786-662j7.mtls-pg` sends a request to `sleep.default`, the Envoy sidecar attached to it will send the request over a plaintext connection (HTTP). We can tweak this as well using [DestinationRules](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/#DestinationRule) which will show up in the `DESTINATION RULE` column.

### [Aside] Policies and Destination Rules

Let's run through how we can change the mTLS mappings between services using Policies and Destination Rules.

Running:

`istio-1.3.2/bin/istioctl authn tls-check sleep-69c766786-662j7.mtls-pg httpbin.mtls-pg.svc.cluster.local`

(we show the mTLS setting with respect to the `httpbin.mtls-pg.svc.cluster.local` service only) yields:

```
HOST:PORT                                  STATUS     SERVER        CLIENT     AUTHN POLICY     DESTINATION RULE
httpbin.mtls-pg.svc.cluster.local:8000     OK         HTTP/mTLS     HTTP       default/         -
```

This tells us that `httpbin.mtls-pg` is running in permissive mode. We can change it to strict mode by applying the following CRD:

```
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "httpbin.mtls-pg"
  namespace: "mtls-pg"
spec:
  targets:
  - name: httpbin
  peers:
  - mtls:
      mode: STRICT
```

Which gives us:

```
HOST:PORT                                  STATUS       SERVER     CLIENT     AUTHN POLICY                DESTINATION RULE
httpbin.mtls-pg.svc.cluster.local:8000     CONFLICT     mTLS       HTTP       httpbin.mtls-pg/mtls-pg     -
```

We've now successfully configured `httpbin.mtls-pg` to **only** accept mTLS connections. The problem here now is that all the pods in the mesh acting as a client to `httpbin.mtls-pg` will only send their request over HTTP which `httpbin.mtls-pg` won't accept, hence the `CONFLICT` status. To fix this, we apply the following CRD:

```
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin.mtls-pg"
  namespace: "mtls-pg"
spec:
  host: httpbin.mtls-pg.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

Which in turn yields:

```
HOST:PORT                                  STATUS     SERVER     CLIENT     AUTHN POLICY                DESTINATION RULE
httpbin.mtls-pg.svc.cluster.local:8000     OK         mTLS       mTLS       httpbin.mtls-pg/mtls-pg     httpbin.mtls-pg/mtls-pg
```

All pods acting as clients to `httpbin.mtls-pg` will now only send their traffic via mTLS. It is important to note though that the previously applied CRD will only affect client requests to `httpbin.mtls-pg` but not to other services, as shown below:

```
HOST:PORT                                  STATUS     SERVER     CLIENT     AUTHN POLICY                DESTINATION RULE
httpbin.mtls-pg.svc.cluster.local:8000     OK         mTLS       mTLS       httpbin.mtls-pg/mtls-pg     httpbin.mtls-pg/mtls-pg
sleep.default.svc.cluster.local:80         OK         HTTP/mTLS  HTTP       default/                    -
```
