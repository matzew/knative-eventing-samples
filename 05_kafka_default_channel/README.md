# Kafka CRD Channel as default and Broker

With the Apache Kafka CRD based channel installed, you can configure Knative to use `KafkaChannels` as default channel implementations. 

## Configuration

To configure the usage of the `KafkaChannels` as default, edit the `default-ch-webhook` Configmap, like:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: default-ch-webhook
  namespace: knative-eventing
data:
  # Configuration for defaulting channels that do not specify CRD implementations.
  default-ch-config: |
    clusterDefault:
      apiVersion: messaging.knative.dev/v1alpha1
      kind: KafkaChannel
```

## Channel CRD

Use the _new_ `channels.messaging.knative.dev` CRD to create a new channel:

```
apiVersion: messaging.knative.dev/v1alpha1
kind: Channel
metadata:
  name: testchannel-one
```

> NOTE: This does not allow you to set any _specific_ details that might be possible for the actual Channel implementation, such as partitions on the Kafka topic.

Now, you can check in Kafka if there is a `testchannel` topic, like:

```
oc -n kafka exec -it my-cluster-kafka-0 -- bin/kafka-topics.sh --zookeeper localhost:2181 --list
```

And see the following result:

```
...
knative-messaging-kafka.default.testchannel-one
...
```

## Kafka based Broker

Inject a new _default_ Broker, that's leveraging the Apache Kafka based channels, like:


```
kubectl label namespace default knative-eventing-injection=enabled
```

This will give you two pods, like:

```
default-broker-filter-64658fc79f-nf596                            1/1     Running     0          15m
default-broker-ingress-ff79755b6-vj9jt                            1/1     Running     0          15m

```


Inside the Apache Kafka cluster you might see two new topics, like:

```
knative-messaging-kafka.default.default-kn2-ingress
knative-messaging-kafka.default.default-kn2-trigger
```

## Demo

Install a `ksvc`, like:

```
kubectl apply -f 000-ksvc.yaml
```

Install a source that publishes to the _default_ broker

```
kubectl apply -f 020-k8s-events.yaml
```

and have a trigger that routes the events to the `ksvc`:

```
kubectl apply -f 030-trigger.yaml
```

### Receive events via Knative

Now you can see the events in the log of the `ksvc`, like:

```
kubectl logs -f broker-display-vr9g8-deployment-64c488b46c-sgv67 -c user-container
```

### Receive events via Kafkacat

Alternative you can connect any Apache Kafka consumer to the exposed `knative-messaging-kafka.default.default-kn2-trigger` topic, like:

```
kafkacat -C -b 192.168.39.86:32002 -t knative-messaging-kafka.default.default-kn2-trigger
```

