# Knative Eventing broker with Apache Kafka backed channels (CRD)

> NOTE: This requires Knative Eventing _AND_ the Apache Kafka channel to be installed already...

## Le setup

Like discussed [here]() setup the right `ServiceAccount` objects:

```
kubectl -n default create serviceaccount eventing-broker-ingress
kubectl -n default create serviceaccount eventing-broker-filter
```

Now, grant them RBAC permissions:

```
kubectl -n default create rolebinding eventing-broker-ingress \
  --clusterrole=eventing-broker-ingress \
  --serviceaccount=default:eventing-broker-ingress
kubectl -n default create rolebinding eventing-broker-filter \
  --clusterrole=eventing-broker-filter \
  --serviceaccount=default:eventing-broker-filter
```

> NOTE: ALL ASSUMES DEFAULT INSTALLATION (e.g. knative-eventing namepsace)

## Create the broker:

Run this:

```
kubectl apply -f 000-kafka-broker.yaml
```

### Test and verify 

Now we can check if the used Apache Kafka cluster has some topics:

```
oc -n kafka exec -it my-cluster-kafka-0 -- bin/kafka-topics.sh --zookeeper localhost:2181 --list
```


and see:

```
knative-messaging-kafka.default.kafka-kn-ingress
knative-messaging-kafka.default.kafka-kn-trigger
```

## Run the demo

Deploy consumer service:

```
kubectl apply -f 010-ksvc.yaml
```

Have a source publishing events to the mesh/broker:

```
kubectl apply -f 020-k8s-events.yaml
```

Finally apply a filter that routes the events to our service:

```
kubectl apply -f 030-trigger.yaml
```

and now see some events in the consumer:

```



```