# Source to Service 

One of the simplest uses cases of Knative eventing is to directly sink the events from the emitting source a Knative Serving Service (`ksvc`).

## Deploy the consuming Knative Serving Service

The first step is the deployment of a Knative Serving service that later acts as the consumer of the events, emitted by the _Knative Eventing Source_:

```
k apply -f 000-ksvc.yaml
```

The above installs a new `ksvc`, called _service-one_, to the `default` namespace. After applying file, we can check some details about our new `ksvc`, by running ` k get ksvc`:

```
NAME               URL                                           LATESTCREATED            LATESTREADY              READY   REASON
service-one0   http://service-one0.default.example.com   service-one0-kfhpm   service-one0-kfhpm   True    
```

We now have a `ksvc` running in the background, waiting for some incoming HTTP requests.

## Connect the Source to the `ksvc`

To be able to have the `ksvc` consume events, we need to hook it up to a _Knative Eventing Source_. In our tutorial we are using the `ApiServerSource`, which will allows an integration of the Kubernetes API server events into Knative.

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
  name: testevents
  namespace: default
spec:
  serviceAccountName: events-sa
  mode: Resource
  resources:
  - apiVersion: v1
    kind: Event
  sink:
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    name: service-one
```

The `spec` of the `ApiServerSource` is referencing the previously created `ServiceAccount`, but more importantly, the `sink` part does reference our `service-one` instance of the initially installed `ksvc`.

Now let's apply it:

```
k apply -f 020-k8s-events.yaml
```

We will now see that a new pod is up and running, representing the `ApiServerSource`, when we run `k get pods`:

```
NAME                                                              READY   STATUS    RESTARTS   AGE
apiserversource-testevents-91f06d1f-cfe6-11e9-9d67-70c21b48r9pt   1/1     Running   0          86m
service-one0-kfhpm-deployment-747c97d7c5-x6hnl                    2/2     Running   0          85s
```

The `apiserversource` pod is now running the `ApiServerSource`, which directly sends its events to the hardwired `ksvc`, our `service-one` installation.

## Show the consumed events

Finally we can now see the consumed Kubernetes API server events in the log of the `service-one0-kfhpm-deployment-747c97d7c5-x6hnl` pod, by running:

```
k logs -f service-one0-kfhpm-deployment-747c97d7c5-x6hnl -c user-container
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

## Conclusion 

We now have seen that one can simply consume events using a _Knative Eventing Source_ that is directly wired to a Knative Serving Service.

While this use-case is simple and easy to implement, it has some limitations, that the _Knative Eventing Source_ is not able to distribute its events to multiple consumers.

NEXT: Channel and multiple consumers.....