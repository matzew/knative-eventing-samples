# Source to Channel with multiple Subscribers 

A more advanced and flexible use case of Knative eventing is to sink the events from the emitting source to a Knative Eventing channel. On the channel we can have multiple subscribers, that are all receiving the _same_ events, for different purposes. 

## Deploy the first consuming Knative Serving Service

The first step is the deployment of one Knative Serving service that later acts as one of the consumers for the events, emitted by the _Knative Eventing Source_:

```
k apply -f 000-ksvc.yaml
```

The above installs a new `ksvc`, called _channel-display0_, to the `default` namespace. After applying the file, we can check some details about our new `ksvc`, by running ` k get ksvc`:

```
NAME               URL                                           LATESTCREATED            LATESTREADY              READY   REASON
default     	   channel-display0-gl82b-deployment-7d5c78fd45-95548   2/2     Running   0          5s
```

We now have a `ksvc` running in the background, waiting for some incoming HTTP requests.

## Create a Channel

Next we need to create a channel and in our case we use the `InMemoryChannel` API:

```
apiVersion: messaging.knative.dev/v1alpha1
kind: InMemoryChannel
metadata:
  name: testchannel
```

> NOTE: Besides `InMemoryChannel` it is also possible to have other channels act as default, such as the `KafkaChannel`.

Now we apply the yaml file:

```
k apply -f 010-channel.yaml
```

Let's see what we got, by running `k get channel`:

```
NAME                                                READY   REASON   HOSTNAME                                           AGE
inmemorychannel.messaging.knative.dev/testchannel   True             testchannel-kn-channel.default.svc.cluster.local   4s
```

We have one entry that shows us our `InMemoryChannel` and its `HOSTNAME`, which represents the HTTP endpoint where the channel is accepting incoming HTTP messages for further delivery.

## Connect the Source to the `InMemoryChannel`

In order to publish events from one _Knative Eventing Source_ to multiple consumers, we need to connect the source to the channel, instead of a single `ksvc`.


### ServiceAccount to run the `ApiServerSource`

Before we can apply the `ApiServerSource` itself, we need to create a `ServiceAccount` that the `ApiServerSource` runs as, since it does require some permissions in order to work with the Kubernetes API server events:

```
k apply -f 010-serviceaccount.yaml
```

### Kubernetes API server events

With the `ServiceAccount` in place we can finally connect our `ApiServerSource` to the `ksvc` so it can consume events. Let's take a look at the file:

```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: ApiServerSource
metadata:
  name: testevents02
  namespace: default
spec:
  serviceAccountName: events-sa
  mode: Resource
  resources:
  - apiVersion: v1
    kind: Event
  sink:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
    name: testchannel
```

The `spec` of the `ApiServerSource` is referencing the previously created `ServiceAccount`, but more importantly, the `sink` part references our `testchannel`.

Now let's apply it:

```
k apply -f 020-k8s-events.yaml
```

We will now see that a new pod is up and running, representing the `ApiServerSource`, when we run `k get pods`:

```
NAME                                                              READY   STATUS    RESTARTS   AGE
apiserversource-testevents-216f6524-d08d-11e9-b515-90d57cb8xf7x   1/1     Running   0          86m
channel-display0-kfhpm-deployment-747c97d7c5-x6hnl                    2/2     Running   0          85s
```

The `apiserversource` pod is now running the `ApiServerSource`, which directly sends its events to the channel, but our consumers are not yet receiving any message.

## Subscribe a Service to the Channel

With the usage of a Channel we have a great level of flexibility, that we can _subscribe_ as many consumers we want to a given channel, like:

```yaml
apiVersion: messaging.knative.dev/v1alpha1
kind: Subscription
metadata:
  name: sub1
spec:
  channel:
    apiVersion: messaging.knative.dev/v1alpha1
    kind: InMemoryChannel
    name: testchannel
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: channel-display0
```

The `Subscription`'s `spec` references two main objects. The `channel` section tells it what Channel is used to subscribe on and the `subscriber` section has an `ObjectRef` to a Knative Serving service. Here we use the one that we had deployed a couple of steps ago.

```
k apply -f 030-subscription.yaml
```

## Show the consumed events

With the subscription in place the messages are now dispatched to the `ksvc` from the used channel. In the case of the `InMemoryChannel` this is done by the `imc-dispatcher` pod, running in the `knative-eventing` namespace.

Now we can finally see the consumed Kubernetes API server events in the log of the `channel-display0-kfhpm-deployment-747c97d7c5-x6hnl` pod, by running:

```
k logs -f channel-display0-kfhpm-deployment-747c97d7c5-x6hnl -c user-container
```

The pod does run two containers, but we are only interested in the `user-container`, running our application via Knative Serving. In the log we now see some Kubernetes API server events, wrapped as CloudEvents:

```
☁️  CloudEvent: valid ✅
Context Attributes,
  SpecVersion: 0.3
  Type: dev.knative.apiserver.resource.add
  Source: https://10.96.0.1:443
  ID: f3a52b07-d1d9-4179-84c2-a5b38406fecd
  Time: 2019-09-05T15:39:02.877250641Z
  DataContentType: application/json
  Extensions: 
    knativehistory: testchannel-kn-channel.default.svc.cluster.local
    subject: /apis/v1/namespaces/default/events/channel-display0-kfhpm-deployment-747c97d7c5-hq6tl.15c194f91805af2e
Transport Context,
  URI: /
  Host: channel-display0.default.svc.cluster.local
  Method: POST
Data,
  {
    "apiVersion": "v1",
    "count": 1,
    "eventTime": null,
    "firstTimestamp": "2019-09-05T15:39:02Z",
    "involvedObject": {
      "apiVersion": "v1",
      "fieldPath": "spec.containers{queue-proxy}",
      "kind": "Pod",
      "name": "channel-display0-kfhpm-deployment-747c97d7c5-hq6tl",
      "namespace": "default",
      "resourceVersion": "13751",
      "uid": "fb0fe720-cff2-11e9-9d67-70c21b40e0b5"
    },
    "kind": "Event",
    "lastTimestamp": "2019-09-05T15:39:02Z",
    "message": "Readiness probe failed: cannot exec in a stopped state: unknown\r\n",
    "metadata": {
      "creationTimestamp": "2019-09-05T15:39:02Z",
      "name": "channel-display0-kfhpm-deployment-747c97d7c5-hq6tl.15c194f91805af2e",
      "namespace": "default",
      "resourceVersion": "14008",
      "selfLink": "/api/v1/namespaces/default/events/channel-display0-kfhpm-deployment-747c97d7c5-hq6tl.15c194f91805af2e",
      "uid": "4981805f-cff3-11e9-9d67-70c21b40e0b5"
    },
    "reason": "Unhealthy",
    "reportingComponent": "",
    "reportingInstance": "",
    "source": {
      "component": "kubelet",
      "host": "minikube"
    },
    "type": "Warning"
  }
```

## More consumers

With one consumer running we now can deploy anothter Knative Serving Service:

```
k apply -f 040-ksvc.yaml
```

and than use that in a _second_ subscription:

```
k apply -f 041-subscription.yaml
```

This will now result in a second `ksvc` that is receiving the _exact_ same events that the first `ksvc` is receiving from its `Subscription`.

## Conclusion 

We now have seen how to distribute events from one _Knative Eventing Source_ to multiple consumers, using the `Subscription` API

While this use-case offeres a greater flexibility to write different services that perform a different processing of the _same_ events, it still has some limitations, that we can not apply any filter of the events that we would like to route to our different services.

NEXT: Broker and Trigger
