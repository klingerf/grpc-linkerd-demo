# gRPC + Linkerd Demo

Hello friends. In this repo you'll find a sample [gRPC](https://grpc.io)
application that can be run with [Linkerd](https://linkerd.io) on
[Kubernetes](https://kubernetes.io). The goal is to demonstrate the operability
features that you gain by running these three technologies together.

The demo is entirely open source, and it leverages some pretty cool projects:

* [Kubernetes](https://github.com/kubernetes)
* [gRPC](https://github.com/grpc)
* [Linkerd](https://github.com/linkerd/linkerd2)
* [Prometheus](https://github.com/prometheus/prometheus), by way of Linkerd
* [Grafana](https://github.com/grafana/grafana), by way of Linkerd
* [Stern](https://github.com/wercker/stern), for tailing container logs
* [Building Blocks](https://github.com/buoyantio/bb), for building a sample app
* [Slow Cooker](https://github.com/buoyantio/slow_cooker), for sending traffic

Check them out when you have the chance.

## Pre-requisites

Before we get started, you'll need to install a few tools, as follows:

1. Setup a working Kubernetes cluster and configure the `kubectl` CLI.
   (see https://kubernetes.io/docs/setup/ for more info)

2. Install the `linkerd` CLI: `curl https://run.linkerd.io/install | sh`
   (see https://linkerd.io/2/getting-started/ for more info)

3. Install the `stern` CLI: https://github.com/wercker/stern#installation

## Install the sample app

Ok, let's get started. Start by installing the sample application defined in
`app.yml`, in the `default` namespace of your Kubernetes cluster:

```
kubectl apply -f app.yml
```

This will create a `traffic` deployment, a `gateway` deployment, and two
backends. The `rest-backend` deployment serves requests over HTTP/1.1, and the
`grpc-backend` deployment serves requests over gRPC (HTTP/2). The topology of
the application looks like this:

![topology](topology.png)

If we tail the logs of the `traffic` pod, we'll see incremental reports of the
traffic that it is sending:

```
stern traffic
```

We can see that it's sending traffic to `http://gateway:7000`:

```
+ traffic-69cf7bc6b9-lc79t › slow-cooker
traffic-69cf7bc6b9-lc79t slow-cooker # sending 60 GET req/s with concurrency=15 to http://gateway:7000 ...
traffic-69cf7bc6b9-lc79t slow-cooker #                      good/b/f t   goal%  min [p50 p95 p99  p999]  max bhash change
traffic-69cf7bc6b9-lc79t slow-cooker 2019-05-02T23:27:05Z     30/5/0 300  11% 5s  41 [107 4615 4643 4643 ] 4643      0 +
traffic-69cf7bc6b9-lc79t slow-cooker 2019-05-02T23:27:10Z    300/0/0 300 100% 5s  11 [ 42  90 172  180 ]  180      0
```

Now let's find out which backend instances are serving this traffic. Start by
looking at the logs of the REST backend:

```
stern rest-backend
```

We'll see that all three instances are actively logging that they are receiving
traffic. Compare that to the gRPC backend:

```
stern grpc-backend
```

We'll see that only a single instance is logging and receiving any traffic.
Let's fix that!

## Running with Linkerd

We can use Linkerd to properly load balance gRPC requests. Start by installing
the Linkerd control plane in your cluster:

```
linkerd install | kubectl apply -f -
```

Verify that the control plane is properly installed by running:

```
linkerd check
```

Once the control plane is installed, we can inject the data plane into our
application by running:

```
linkerd inject app.yml | kubectl apply -f -
```

Now when we look at the gRPC backend logs, we'll see that traffic is being
balanced across all three instances:

```
stern grpc-backend
```

Great!

## Viewing stats

With Linkerd in the mix, we can see a lot more stats about our traffic, as well.
First off, we can see basic stats about how each deployment is performing:

```
linkerd stat deploy -owide
```

That will show us stats like:

```
NAME           MESHED   SUCCESS       RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99   TCP_CONN   READ_BYTES/SEC   WRITE_BYTES/SEC
gateway           1/1   100.00%   60.1rps          80ms         181ms         200ms         19       15215.6B/s        17077.3B/s
grpc-backend      3/3   100.00%   60.0rps           6ms          29ms          83ms          6       14633.2B/s        14238.9B/s
rest-backend      3/3   100.00%   60.2rps           6ms          27ms          51ms         19       19206.7B/s        24821.2B/s
traffic           1/1         -         -             -             -             -          -                -                 -
```

There is a bunch of important data here. Since the Linkerd proxy is performing
protocol detection, it's able to tell us information about the performance of
individual requests, such as request rate, success rate, and percentile
latencies. It's also able to report connection-level stats about the TCP
connections themselves.

We can also view stats for each pod in the backend deployments, to see how
well Linkerd is balancing traffic to those pods:

```
linkerd stat pods --from deploy/gateway
```

That will show:

```
NAME                            MESHED   SUCCESS       RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99   TCP_CONN
grpc-backend-65c78674f8-d89v5      1/1   100.00%   18.1rps          24ms          81ms        1375ms          0
grpc-backend-65c78674f8-llqwd      1/1   100.00%   19.9rps          24ms          81ms        1527ms          0
grpc-backend-65c78674f8-ppzqd      1/1   100.00%   21.3rps          28ms          86ms          97ms          0
rest-backend-7c9fb95f55-2q8zj      1/1   100.00%   18.0rps          24ms        1262ms        1853ms          0
rest-backend-7c9fb95f55-5cqqp      1/1   100.00%   20.5rps          27ms          81ms          98ms          0
rest-backend-7c9fb95f55-8x79j      1/1   100.00%   20.8rps          28ms          82ms          96ms          0
```

We can also inspect individual requests with Linkerd's `tap` command:

```
linkerd tap deploy/grpc-backend
```

That will show very granular data about each request proxied.

And finally, all of this data is also available in Linkerd's web UI, by running:

```
linkerd dashboard
```

## Adding service profiles

Next we'll explore using
[service profiles](https://linkerd.io/2/features/service-profiles/) to enable
per-endpoint metrics and assuming control of timeouts for clients calling the
`grpc-backend` service.

A service profile can be created directly from the protobuf definition for our
gRPC backend, by running:

```
linkerd profile --proto grpc-backend.proto grpc-backend | kubectl apply -f -
```

With the profile in place, we can use the `routes` command to see data about how
each route is performing:

```
linkerd routes svc/grpc-backend
```

That will display:

```
ROUTE              SERVICE   SUCCESS       RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
[DEFAULT]     grpc-backend     0.00%    0.0rps           0ms           0ms           0ms
theFunction   grpc-backend   100.00%   58.8rps           6ms          30ms          58ms
```

It's not super exciting because our protobuf definition only has one route
(`theFunction`), but if it had more, they'd show up here.

Service profiles can also be use to provide per-route configuration. For
instance, let's set a timeout on requests to the `grpc-gateway` service, by
editing the service profile we just created:

```
kubectl edit sp/grpc-backend.default.svc.cluster.local
```

And adding a `timeout` field as follows:

```
spec:
  routes:
  - condition:
      method: POST
      pathRegex: /buoyantio\.bb\.TheService/theFunction
    name: theFunction
    timeout: 75ms
```

That will timeout all requests taking longer than 75 milliseconds. And sure
enough, if we now look at stats for requests to the `gateway` deployment (which
calls the `grpc-backend` deployment), we can see that success rate has fallen:

```
linkerd stat deploy/gateway
```

That will display:

```
NAME      MESHED   SUCCESS       RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99   TCP_CONN
gateway      1/1    95.37%   59.4rps          91ms         197ms         382ms         16
```

We can improve success rate by having the `traffic` deployment retry requests to
the `gateway` service, by adding a service profile for the `gateway` service.
Similar to how we created a service profile for the `grpc-backend` service using
protobuf above, we can also create a service profile for the `gateway` service
using a swagger definition, which is defined in the `gateway.swagger` file. Run:

```
linkerd profile --open-api gateway.swagger gateway | kubectl apply -f -
```

Edit the service profile to indicate that the `gateway` service's "GET /" route
is retryable:

```
kubectl edit sp/gateway.default.svc.cluster.local
```

Add an `isRetryable` field as follows:

```
spec:
  routes:
  - condition:
      method: GET
      pathRegex: /
    name: GET /
    isRetryable: true
```

Give the service profile a few seconds to propagate, and then verify that it
fixes success rate for traffic to the `gateway` service:

```
linkerd routes deploy/traffic --to svc/gateway -o wide
```

That will display:

```
ROUTE       SERVICE   EFFECTIVE_SUCCESS   EFFECTIVE_RPS   ACTUAL_SUCCESS   ACTUAL_RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
GET /       gateway             100.00%         59.0rps           92.86%      63.5rps          92ms         286ms        1457ms
[DEFAULT]   gateway               0.00%          0.0rps            0.00%       0.0rps           0ms           0ms           0ms
```

In this output, we can see that the actual success rate of requests to the
`gateway` service is ~93%, but with retries enabled, the effective success rate
is 100%. This means that, from the `traffic` deployment's perspective, it's
requests are succeeding 100% of the time, since Linkerd is automatically
retrying failed request in the background.

## Learn more

For more information about Linkerd and gRPC, check out the following blog posts:

- https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/
- https://kubernetes.io/blog/2018/09/18/hands-on-with-linkerd-2.0/
