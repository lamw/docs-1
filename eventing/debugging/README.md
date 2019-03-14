# Debugging Knative Eventing

This is an evolving document on how to debug a non-working Knative Eventing
setup.

## Audience

This document is intended for people that are familiar with
[Knative Eventing](../README.md)'s object model. You don't need to be an expert,
but do need to know roughly how things fit together.

## Version

This document works with
[Eventing 0.3](https://github.com/knative/eventing/releases/tag/v0.3.0) and
[Eventing Sources 0.3](https://github.com/knative/eventing-sources/releases/tag/v0.3.0).

## Prerequisites

1. Setup [Knative Eventing and Eventing-Sources](../README.md).

## Example

This guide uses an example consisting of an Event Source sending events to a
function.

![src -> chan -> sub -> svc -> fn](ExampleModel.png)

See [example.yaml](example.yaml) for the entire YAML. For any commands in this
guide to work, you must apply [example.yaml](example.yaml):

```shell
kubectl apply --filename example.yaml
```

## Triggering Events

Knative events will occur whenever a Kubernetes
[`Event`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#event-v1-core)
occurs in the `knative-debug` namespace. We can cause this to occur with the
following commands:

```shell
kubectl --namespace knative-debug run to-be-deleted --image=image-that-doesnt-exist --restart=Never
# 5 seconds is arbitrary. We want K8s to notice that the Pod needs to be scheduled and generate at least one event.
sleep 5
kubectl --namespace knative-debug delete pod to-be-deleted
```

Then we can see the Kubernetes `Event`s (note that these are not Knative
events!):

```shell
kubectl --namespace knative-debug get events
```

This should produce output along the lines of:

```shell
LAST SEEN   FIRST SEEN   COUNT     NAME                             KIND      SUBOBJECT                        TYPE      REASON                   SOURCE                                         MESSAGE
20s         20s          1         to-be-deleted.157aadb9f376fc4e   Pod                                        Normal    Scheduled                default-scheduler                              Successfully assigned knative-debug/to-be-deleted to gke-kn24-default-pool-c12ac83b-pjf2
```

## Where are my events?

You've applied [example.yaml](example.yaml) and you are inspecting `fn`'s logs:

```shell
kubectl --namespace knative-debug logs -l app=fn -c user-container
```

But you don't see any events arrive. Where is the problem?

### Control Plane

We will first check the control plane, to ensure everything should be working
properly.

#### Resources

The first thing to check are all the created resources, do their statuses
contain `ready` true?

We will attempt to determine why from the most basic pieces out:

1. `fn` - The `Deployment` has no dependencies inside Knative.
1. `svc` - The `Service` has no dependencies inside Knative.
1. `chan` - The `Channel` depends on its backing `ClusterChannelProvisioner` and
   somewhat depends on `sub`.
1. `src` - The `Source` depends on `chan`.
1. `sub` - The `Subscription` depends on both `chan` and `svc`.

##### `fn`

```shell
kubectl --namespace knative-debug get deployment fn -o jsonpath='{.status.availableReplicas}'
```

We want to see `1`. If you don't, then you need to debug the `Deployment`. Is
there anything obviously wrong mentioned in the `status`?

```shell
kubectl --namespace knative-debug get deployment fn --output yaml
```

If it is not obvious what is wrong, then you need to debug the `Deployment`,
which is out of scope of this document.

Verify that the `Pod` is `Ready`:

```shell
kubectl --namespace knative-debug get pod -l app=fn -o jsonpath='{.items[*].status.conditions[?(@.type == "Ready")].status}'
```

This should return `True`. If it doesn't, then try to debug the `Deployment` using the [Kubernetes Application Debugging](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application-introspection/) guide.

##### `svc`

```shell
kubectl --namespace knative-debug get service svc
```

We just want to ensure this exists and has the correct name. If it doesn't
exist, then you probably need to re-apply [example.yaml](example.yaml).

Verify it points at the expected pod.

```shell
svcLabels=$(kubectl --namespace knative-debug get service svc -o go-template='{{range $k, $v := .spec.selector}}{{ $k }}={{ $v }},{{ end }}' | sed 's/.$//' )
kubectl --namespace knative-debug get pods -l $svcLabels
```

This should return a single Pod, which if you inspect is the one generated by
`fn`.

##### `chan`

`chan` uses the
[`in-memory-channel`](https://github.com/knative/eventing/tree/master/config/provisioners/in-memory-channel)
as its `ClusterChannelProvisioner`. This is a very basic provisioner and has few
failure modes that will be exhibited in `chan`'s `status`.

```shell
kubectl --namespace knative-debug get channel chan -o jsonpath='{.status.conditions[?(@.type == "Ready")].status}'
```

This should return `True`. If it doesn't, get the full resource:

```shell
kubectl --namespace knative-debug get channel chan --output yaml
```

If `status` is completely missing, it implies that something is wrong with the
`in-memory-channel` controller. See [Channel Controller](#channel-controller).

Next verify that `chan` is addressable:

```shell
kubectl --namespace knative-debug get channel chan -o jsonpath='{.status.address.hostname}'
```

This should return a URI, likely ending in '.cluster.local'. If it doesn't, then
it implies that something went wrong during reconcilation. See
[Channel Controller](#channel-controller).

We will verify that the two resources that the `chan` creates exist and are
`Ready`.

###### `Service`

`chan` creates a K8s `Service`.

```shell
kubectl --namespace knative-debug get service -l provisioner=in-memory-channel,channel=chan
```

It's spec is completely unimportant, as Istio will ignore it. It just needs to
exist so that `src` can send events to it. If it doesn't exist, it implies that
something went wrong during `chan` reconciliation. See
[Channel Controller](#channel-controller).

###### `VirtualService`

`chan` creates a `VirtualService` which redirects its hostname to the
`in-memory-channel` dispatcher.

```shell
kubectl --namespace knative-debug get virtualservice -l provisioner=in-memory-channel,channel=chan -o custom-columns='HOST:.spec.hosts[0],DESTINATION:.spec.http[0].route[0].destination.host'
```

Verify that

1. `HOST` is the same as the hostname returned by in `chan`'s
   `status.address.hostname`.
1. `DESTINATION` is
   `in-memory-channel-dispatcher.knative-eventing.svc.cluster.local`.

If either of those is not accurate, then it implies that something went wrong
during `chan` reconciliation. See [Channel Controller](#channel-controller).

##### `src`

`src` is a
[`KubernetesEventSource`](https://github.com/knative/eventing-sources/blob/master/pkg/apis/sources/v1alpha1/kuberneteseventsource_types.go),
which creates an underlying
[`ContainerSource`](https://github.com/knative/eventing-sources/blob/master/pkg/apis/sources/v1alpha1/containersource_types.go).

First we will verify that `src` is writing to `chan`.

```shell
kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.spec.sink}'
```

Which should return
`map[apiVersion:eventing.knative.dev/v1alpha1 kind:Channel name:chan]`. If it
doesn't, then `src` was setup incorrectly and its `spec` needs to be fixed.
Fixing should be as simple as updating its `spec` to have the correct `sink`
(see [example.yaml](example.yaml)).

Now that we know `src` is sending to `chan`, let's verify that it is `Ready`.

```shell
kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.status.conditions[?(.type == "Ready")].status}'
```

This should return `True`. If it doesn't, then we need to investigate why. First
we will look at the owned `ContainerSource` that underlies `src`, and if that is
not fruitful, look at the [Source Controller](#source-controller).

##### ContainerSource

`src` is backed by a `ContainerSource` resource.

Is the `ContainerSource` `Ready`?

```shell
srcUID=$(kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.metadata.uid}')
kubectl --namespace knative-debug get containersource -o jsonpath="{.items[?(.metadata.ownerReferences[0].uid == '$srcUID')].status.conditions[?(.type == 'Ready')].status}"
```

That should be `True`. If it is, but `src` is not `Ready`, then that implies the
problem is in the [Source Controller](#source-controller).

If `ContainerSource` is not `Ready`, then we need to look at its entire
`status`:

```shell
srcUID=$(kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.metadata.uid}')
containerSourceName=$(kubectl --namespace knative-debug get containersource -o jsonpath="{.items[?(.metadata.ownerReferences[*].uid == '$srcUID')].metadata.name}")
kubectl --namespace knative-debug get containersource $containerSourceName --output yaml
```

The most useful condition (when `Ready` is not `True`), is `Deployed`.

```shell
srcUID=$(kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.metadata.uid}')
containerSourceName=$(kubectl --namespace knative-debug get containersource -o jsonpath="{.items[?(.metadata.ownerReferences[*].uid == '$srcUID')].metadata.name}")
kubectl --namespace knative-debug get containersource $containerSourceName -o jsonpath='{.status.conditions[?(.type == "Deployed")].message}'
```

You should see something like `Updated deployment src-xz59f-hmtkp`. Let's see
the health of the `Deployment` that `ContainerSource` created (named in the
message, but we will get it directly in the following command):

```shell
srcUID=$(kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.metadata.uid}')
containerSourceUID=$(kubectl --namespace knative-debug get containersource -o jsonpath="{.items[?(.metadata.ownerReferences[*].uid == '$srcUID')].metadata.uid}")
deploymentName=$(kubectl --namespace knative-debug get deployment -o jsonpath="{.items[?(.metadata.ownerReferences[*].uid == '$containerSourceUID')].metadata.name}")
kubectl --namespace knative-debug get deployment $deploymentName --output yaml
```

If this is unhealthy, then it should tell you why. E.g.
`'pods "src-xz59f-hmtkp-7bd4bc6964-" is forbidden: error looking up service account knative-debug/events-sa: serviceaccount "events-sa" not found'`.
Fix any errors so that it the `Deployment` is healthy.

If the `Deployment` is healthy, but the `ContainerSource` isn't, that implies
something went wrong in
[ContainerSource Controller](#containersource-controller).

#### `sub`

`sub` is a `Subscription` from `chan` to `fn`.

Verify that `sub` is `Ready`:

```shell
kubectl --namespace knative-debug get subscription sub -o jsonpath='{.status.conditions[?(.type == "Ready")].status}'
```

This should return `True`. If it doesn't then, look at all the status entries.

```shell
kubectl --namespace knative-debug get subscription sub --output yaml
```

#### Controllers

Each of the resources has a Controller that is watching it. As of today, they
tend to do a poor job of writing failure status messages and events, so we need
to look at the Controller's logs.

##### Deployment Controller

The Kubernetes Deployment Controller, controlling `fn`, is out of scope for this
document.

##### Service Controller

The Kubernetes Service Controller, controlling `svc`, is out of scope for this
document.

##### Channel Controller

There is not a single `Channel` Controller. Instead, there is a single
Controller for each `ClusterChannelProvisioner`. `chan` uses the
`in-memory-channel` `ClusterChannelProvisioner`, whose Controller is:

```shell
kubectl --namespace knative-eventing get pod -l clusterChannelProvisioner=in-memory-channel,role=controller --output yaml
```

See its logs with:

```shell
kubectl --namespace knative-eventing logs -l clusterChannelProvisioner=in-memory-channel,role=controller
```

Pay particular attention to any lines that have a logging level of `warning` or
`error`.

##### Source Controller

Each Source will have its own Controller. `src` is a `KubernetesEventSource`, so
its Controller is:

```shell
kubectl --namespace knative-sources get pod -l control-plane=controller-manager
```

This is actually a single binary that runs multiple Source Controllers,
importantly including [ContainerSource Controller](#containersource-controller).

The `KubernetesEventSource` is fairly simple, as it delegates all functionality
to an underlying [ContainerSource](#containersource), so there is likely no
useful information in its logs. Instead more useful information is likely in the
[ContainerSource Controller](#containersource-controller)'s logs. If you want to
look at `KubernetesEventSource` Controller's logs anyway, they can be see with:

```shell
kubectl --namespace knative-sources logs -l control-plane=controller-manager
```

###### ContainerSource Controller

The `ContainerSource` Controller is run in the same binary as some other Source
Controllers. It is:

```shell
kubectl --namespace knative-sources get pod -l control-plane=controller-manager
```

View its logs with:

```shell
kubectl --namespace knative-sources logs -l control-plane=controller-manager
```

Pay particular attention to any lines that have a logging level of `warning` or
`error`.

##### Subscription Controller

The `Subscription` Controller controls `sub`. It attempts to resolve the
addresses that a `Channel` should send events to, and once resolved, inject
those into the `Channel`'s `spec.subscribable`.

```shell
kubectl --namespace knative-eventing get pod -l app=eventing-controller
```

View its logs with:

```shell
kubectl --namespace knative-eventing logs -l app=eventing-controller
```

Pay particular attention to any lines that have a logging level of `warning` or
`error`.

### Data Plane

The entire [Control Plane](#control-plane) looks healthy, but we're still not
getting any events. Now we need to investigate the data plane.

The Knative event takes the following path:

1. Event is generated by `src`.

   - In this case, it is caused by having a Kubernetes `Event` trigger it, but
     as far as Knative is concerned, the `Source` is generating the event denovo
     (from nothing).

1. `src` is POSTing the event to `chan`'s address,
   `chan-channel-45k5h.knative-debug.svc.cluster.local`.

1. `src`'s Istio proxy is intercepting the request, seeing that the Host matches
   a `VirtualService`. The request's Host is rewritten to
   `chan.knative-debug.channels.cluster.local` and sent to the
   [Channel Dispatcher](#channel-dispatcher),
   `in-memory-channel-dispatcher.knative-eventing.svc.cluster.local`.

1. The Channel Dispatcher receives the request and introspects the Host header
   to determine which `Channel` it corresponds to. It sees that it corresponds
   to `knative-debug/chan` so forwards the request to the subscribers defined in
   `sub`, in particular `svc`, which is backed by `fn`.

1. `fn` receives the request and logs it.

We will investigate components in the order in which events should travel.

#### `src`

Events should be generated at `src`. First let's look at the `Pod`s logs:

```shell
srcUID=$(kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.metadata.uid}')
containerSourceName=$(kubectl --namespace knative-debug get containersource -o jsonpath="{.items[?(.metadata.ownerReferences[*].uid == '$srcUID')].metadata.name}")
kubectl --namespace knative-debug logs -l source=$containerSourceName -c source
```

Note that a few log lines within the first ~15 seconds of the `Pod` starting
like the following are fine. They represent the time waiting for the Istio proxy
to start. If you see these more than a few seconds after the `Pod` starts, then
something is wrong.

```shell
E0116 23:59:40.033667       1 reflector.go:205] github.com/knative/eventing-sources/pkg/adapter/kubernetesevents/adapter.go:73: Failed to list *v1.Event: Get https://10.51.240.1:443/api/v1/namespaces/kna tive-debug/events?limit=500&resourceVersion=0: dial tcp 10.51.240.1:443: connect: connection refused
E0116 23:59:41.034572       1 reflector.go:205] github.com/knative/eventing-sources/pkg/adapter/kubernetesevents/adapter.go:73: Failed to list *v1.Event: Get https://10.51.240.1:443/api/v1/namespaces/kna tive-debug/events?limit=500&resourceVersion=0: dial tcp 10.51.240.1:443: connect: connection refused
```

The success message is `debug` level, so we don't expect to see anything. If you
see lines with a logging level of `error`, look at their `msg`. For example:

```shell
"msg":"[404] unexpected response \"\""
```

Which means that `src` correctly got the Kubernetes `Event` and tried to send it
to `chan`, but failed to do so. In this case, the response code was a 404. We
will look at the Istio proxy's logs to see if we can get any further
information:

```shell
srcUID=$(kubectl --namespace knative-debug get kuberneteseventsource src -o jsonpath='{.metadata.uid}')
containerSourceName=$(kubectl --namespace knative-debug get containersource -o jsonpath="{.items[?(.metadata.ownerReferences[*].uid == '$srcUID')].metadata.name}")
kubectl --namespace knative-debug logs -l source=$containerSourceName -c istio-proxy
```

We see lines like:

```shell
[2019-01-17T17:16:11.898Z] "POST / HTTP/1.1" 404 NR 0 0 0 - "-" "Go-http-client/1.1" "4702a818-11e3-9e15-b523-277b94598101" "chan-channel-45k5h.knative-debug.svc.cluster.local" "-"
```

These are lines emitted by [Envoy](https://www.envoyproxy.io). The line is
documented as Envoy's
[Access Logging](https://www.envoyproxy.io/docs/envoy/latest/configuration/access_log).
That's odd, we already verified that there is a
[`VirtualService`](#virtualservice) for `chan`. In fact, we don't expect to see
`chan-channel-45k5h.knative-debug.svc.cluster.local` at all, it should be
replaced with `chan.knative-debug.channels.cluster.local`. We keep looking in
the same Istio proxy logs and see:

```shell
 [2019-01-16 23:59:41.408][23][warning][config] bazel-out/k8-opt/bin/external/envoy/source/common/config/_virtual_includes/grpc_mux_subscription_lib/common/config/grpc_mux_subscription_impl.h:70] gRPC     config for type.googleapis.com/envoy.api.v2.RouteConfiguration rejected: Only unique values for domains are permitted. Duplicate entry of domain chan.knative-debug.channels.cluster.local
```

This shows that the [`VirtualService`](#virtualservice) created for `chan`,
which tries to map two hosts,
`chan-channel-45k5h.knative-debug.svc.cluster.local` and
`chan.knative-debug.channels.cluster.local`, is not working. The most likely
cause is duplicate `VirtualService`s that all try to rewrite those hosts. Look
at all the `VirtualService`s in the namespace and see what hosts they rewrite:

```shell
kubectl --namespace knative-debug get virtualservice -o custom-columns='NAME:.metadata.name,HOST:.spec.hosts[*]'
```

In this case, we see:

```shell
NAME                 HOST
chan-channel-38x5a   chan-channel-45k5h.knative-debug.svc.cluster.local,chan.knative-debug.channels.cluster.local
chan-channel-8dc2x   chan-channel-45k5h.knative-debug.svc.cluster.local,chan.knative-debug.channels.cluster.local
```

```
Note: This shouldn't happen normally. It only happened here because I had local edits to the Channel controller and created a bug. If you see this with any released Channel Controllers, open a bug with all relevant information (Channel Controller info and YAML of all the VirtualServices).
```

Both are owned by `chan`. Deleting both, causes the
[Channel Controller](#channel-controller) to recreate the correct one. After
deleting both, a single new one is created (same command as above):

```shell
NAME                 HOST
chan-channel-9kbr8   chan-channel-45k5h.knative-debug.svc.cluster.local,chan.knative-debug.channels.cluster.local
```

After [forcing a Kubernetes event to occur](#triggering-events), the Istio proxy
logs now have:

```shell
[2019-01-17T18:04:07.571Z] "POST / HTTP/1.1" 202 - 795 0 1 1 "-" "Go-http-client/1.1" "ba36be7e-4fc4-9f26-83bd-b1438db730b0" "chan.knative-debug.channels.cluster.local" "10.48.1.94:8080"
```

Which looks correct. Most importantly, the return code is now 202 Accepted. In
addition, the request's Host is being correctly rewritten to
`chan.knative-debug.channels.cluster.local`.

#### Channel Dispatcher

The Channel Dispatcher is the component that receives POSTs pushing events into
`Channel`s and then POSTs to subscribers of those `Channel`s when an event is
received. For the `in-memory-channel` used in this example, there is a single
binary that handles both the receiving and dispatching sides for all
`in-memory-channel` `Channel`s.

First we will inspect the Dispatcher's logs to see if it is anything obvious:

```shell
kubectl --namespace knative-eventing logs -l clusterChannelProvisioner=in-memory-channel,role=dispatcher -c dispatcher
```

Ideally we will see lines like:

```shell
{"level":"info","ts":1547752472.9581263,"caller":"provisioners/message_receiver.go:116","msg":"Received request for chan.knative-debug.channels.cluster.local"}
{"level":"info","ts":1547752472.9582398,"caller":"provisioners/message_dispatcher.go:106","msg":"Dispatching message to http://svc.knative-debug.svc.cluster.local/"}
```

Which shows that the request is being received and then sent to `svc`, which is
returning a 2XX response code (likely 200, 202, or 204).

However if we see something like:

```shell
{"level":"info","ts":1547752478.5898774,"caller":"provisioners/message_receiver.go:116","msg":"Received request for chan.knative-debug.channels.cluster.local"}
{"level":"info","ts":1547752478.58999,"caller":"provisioners/message_dispatcher.go:106","msg":"Dispatching message to http://svc.knative-debug.svc.cluster.local/"}
{"level":"error","ts":1547752478.6035335,"caller":"fanout/fanout_handler.go:108","msg":"Fanout had an error","error":"Unable to complete request Post http://svc.knative-debug.svc.cluster.local/: EOF","stacktrace":"github.com/knative/eventing/pkg/sidecar/fanout.(*Handler).dispatch\n\t/usr/local/google/home/harwayne/go/src/github.com/knative/eventing/pkg/sidecar/fanout/fanout_handler.go:108\ngithub.com/knative/eventing/pkg/sidecar/fanout.createReceiverFunction.func1\n\t/usr/local/google/home/harwayne/go/src/github.com/knative/eventing/pkg/sidecar/fanout/fanout_handler.go:86\ngithub.com/knative/eventing/pkg/provisioners.(*MessageReceiver).HandleRequest\n\t/usr/local/google/home/harwayne/go/src/github.com/knative/eventing/pkg/provisioners/message_receiver.go:132\ngithub.com/knative/eventing/pkg/sidecar/fanout.(*Handler).ServeHTTP\n\t/usr/local/google/home/harwayne/go/src/github.com/knative/eventing/pkg/sidecar/fanout/fanout_handler.go:91\ngithub.com/knative/eventing/pkg/sidecar/multichannelfanout.(*Handler).ServeHTTP\n\t/usr/local/google/home/harwayne/go/src/github.com/knative/eventing/pkg/sidecar/multichannelfanout/multi_channel_fanout_handler.go:128\ngithub.com/knative/eventing/pkg/sidecar/swappable.(*Handler).ServeHTTP\n\t/usr/local/google/home/harwayne/go/src/github.com/knative/eventing/pkg/sidecar/swappable/swappable.go:105\nnet/http.serverHandler.ServeHTTP\n\t/usr/lib/google-golang/src/net/http/server.go:2740\nnet/http.(*conn).serve\n\t/usr/lib/google-golang/src/net/http/server.go:1846"}
```

Then we know there was a problem posting to
`http://svc.knative-debug.svc.cluster.local/`.

TODO Finish this section. Especially after the Channel Dispatcher emits K8s
events about failures.

#### `fn`

TODO Fill in this section.

See `fn`'s Istio proxy logs:

```shell
kubectl --namespace knative-debug logs -l app=fn -c istio-proxy
```

# TODO Finish the guide.