# No TLS and No Auth

This is the simplest of the examples, where the Platform will be deployed without TLS (on-the-wire encryption) nor user authentication.

To deploy the platform run:

```bash
kubectl apply -f cp-platform.yaml
```

For each service you can perform the following tests...

## Test Kafka Produce and Consume

Descriptor file `test-kafka-app.yaml` creates a topic named `app-topic`, a secret names `kafka-app-config` with property file to connect internally (to K8s) to kafka (see [internal-client.properties](internal-client.properties)), and two pods one that will produce 10.000 records in 1 second interval and another pod that will consume from the same topic.

```bash
kubectl apply -f test-kafka-app.yaml
```

You can follow the logs of each pod once running to see the producer and consumer apps in action

```bash
kubectl logs -f producer-app

kubectl logs -f consumer-app
```

To tear down the app run the code below, but you can leave it running will you test Confluent Control Center (C3)

```bash
kubectl delete -f test-kafka-app.yaml
```

## Test SR (Schema Registry)

This is a simple check to confirm the REST endpoint works

```bash
kubectl exec -it kafka-0 -- curl -k https://schemaregistry.confluent.svc.cluster.local:8081/schemas/types
```

## Test Connect

This is a simple check to confirm the REST endpoint works

```bash
kubectl exec -it kafka-0 -- curl -k https://connect.confluent.svc.cluster.local:8083/
```

## Test ksqlDB

This is a simple check to confirm the REST endpoint works

```bash
kubectl exec -it kafka-0 -- curl -k https://ksqldb.confluent.svc.cluster.local:8088/info
```

## Test Confluent Control Center (C3)

You need to port-forward port 9021 from C3 pod. Either using the [confluent kubectl plugin](https://docs.confluent.io/operator/current/co-deploy-cfk.html#install-confluent-plugin).

```bash
## This will also open your browser automatically
kubectl confluent dashboard controlcenter
```

Or with kubectl port-foward

```bash
kubectl port-forward controlcenter-0 9021:9021 -n confluent

## Open your browser and navigate to http://localhost:9021
```

## Tear down the platform

```
kubectl delete -f cp-platform.yaml
```
