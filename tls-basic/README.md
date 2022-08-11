# (K8s Internal) TLS and Basic/Plain Auth

This exercise add `SASL/PLAIN` authentication to kafka and `BASIC` for the HTTP-based services, to the [tls-noauth](../tls-noauth/) excercise.

For Kafka Resoruce you will notice the following addition that indicates the secret containing a json file with the allowed users. Also note that authentication is enforced at the "listener".

```yaml
spec:
  ...
  listeners:
    internal:
      authentication:
        type: plain
        jaasConfig:
          secretRef: kafka-internal-users-jaas
      tls:
        enabled: true
```

For other resources, a similar approach is followed, but authentication applies to the whole service.

```yaml
spec:
  ...
  authentication:
    type: basic
    basic:
      secretRef: basic-users-sr
```

For the filenames inside the secrets and their format, refer to the [CFK Authentication Documentation](https://docs.confluent.io/operator/current/co-authenticate.html).

**NOTE:** For simplicity we will use `mtls` for zookeeper<>kafka authentication.

---

## TLS

We will use the same approach as in [tls-noauth](../tls-noauth/) excercise, we will create a CA signer certificate and key, and let CFK create the certificates for each service.

```bash
mkdir generated

openssl genrsa -out generated/InternalCAkey.pem 2048

openssl req -x509 -new -nodes \
  -key generated/InternalCAkey.pem \
  -days 365 \
  -out generated/InternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=InternalCA"

kubectl create secret tls ca-pair-sslcerts \
--cert=generated/InternalCAcert.pem \
--key=generated/InternalCAkey.pem \
--namespace confluent
```

---

## Service Credentials (Secrets)

For each service we will prepare a secret with a list of allowed users, including "service" user, also secrets with the login/password that each service will use to authenticate among them.

The secrets need are summarized in the following table, additional columns have been provided to help setting up the commands to create them

| Secret | Secret filename | Purpose | Content |
|---|---|---|---|
| kafka-internal-users | plain-users.json | Allowed users to Kafka | [file](secrets/kafka-internal-users.json) |
| basic-mr-kafka-secret | plain.txt | user/pass to Kafka for metrics | [file](secrets/basic-mr-kafka-secret.txt) |
| basic-users-sr | basic.txt | Allowed users to SR | [file](secrets/basic-users-sr-withrole.txt) |
| basic-users-connect | basic.txt | Allowed users to Connect | [file](secrets/basic-users-connect.txt) |
| basic-users-ksqldb | basic.txt | Allowed users to ksqldb | [file](secrets/basic-users-ksqldb-withrole.txt) |
| basic-users-c3 | basic.txt | Allowed users to C3 | [file](secrets/basic-users-c3-withrole.txt) |
| basic-sr-kafka-secret | plain.txt | user-pass to Kafka for SR | [file](secrets/basic-sr-kafka-secret.txt) |
| basic-connect-kafka-secret | plain.txt | user-pass to Kafka for Connect | [file](secrets/basic-connect-kafka-secret.txt) |
| basic-ksqldb-kafka-secret | plain.txt | user-pass to Kafka for ksqldb | [file](secrets/basic-ksqldb-kafka-secret.txt) |
| basic-ksqldb-sr-secret | plain.txt | user-pass to SR for ksqldb | [file](secrets/basic-ksqldb-sr-secret.txt) |
| basic-c3-kafka-secret | plain.txt | user-pass to Kafka for C3 | [file](secrets/basic-c3-kafka-secret.txt) |
| basic-c3-sr-secret | plain.txt | user-pass to SR for C3 | [file](secrets/basic-c3-sr-secret.txt) |
| basic-c3-connect-secret | plain.txt | user-pass to Connect for C3 | [file](secrets/basic-c3-connect-secret.txt) |
| basic-c3-ksqldb-secret | plain.txt | user-pass to ksqldb for C3 | [file](secrets/basic-c3-ksqldb-secret.txt) |

Using the above table you can create the needed secrets with the command below

```bash
kubectl create secret generic <secret> \
 --from-file=<secret_filename>=<file> \
 --namespace confluent
```

For this exercise the secret contents are available in the [secrets](secrets/) folder and for "deployment" convenience a `secret.yml` has been prepared nesting a similar command like the above. See [secrest/README.md](secrets/README.md)

```bash
kubectl apply -f secrets/secrets.yml
```

---

## Deploy the platform

To deploy the platform run:

```bash
kubectl apply -f cp-platform.yaml
```

---

For each service you can perform the following tests...

## Test Kafka Produce and Consume

Descriptor file `test-kafka-app.yaml` creates a topic named `app-topic`, a secret named `kafka-client-tls-properties` with property file to connect internally (to K8s) to kafka (see [internal-client.properties](internal-client.properties)), and two pods one that will produce 10.000 records in 1 second interval and another pod that will consume from the same topic.

```bash
kubectl apply -f test-kafka-app.yaml
```

You can follow the logs of each pod once running to see the producer and consumer apps in action

```bash
kubectl logs -f producer-app

kubectl logs -f consumer-app
```

To tear down the app run the code below, but you can leave it running until you test Confluent Control Center (C3) below

```bash
kubectl delete -f test-kafka-app.yaml
```

## Test SR (Schema Registry)

This is a simple check to confirm the REST endpoint works

```bash
## CHECK GENERATED CERTIFICATE
kubectl exec -it kafka-0 -- \
  openssl s_client -connect schemaregistry.confluent.svc.cluster.local:8081 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -E '(Issuer: | DNS:)'

## CHECK THE SERVICE - NO USER
kubectl exec -it kafka-0 -- curl -k https://schemaregistry.confluent.svc.cluster.local:8081/schemas/types

## CHECK THE SERVICE - AUTHENTICATING
kubectl exec -it kafka-0 -- curl -k -u sr-user:sr-password https://schemaregistry.confluent.svc.cluster.local:8081/schemas/types
```

## Test Connect

This is a simple check to confirm the REST endpoint works

```bash
## CHECK GENERATED CERTIFICATE
kubectl exec -it kafka-0 -- \
  openssl s_client -connect connect.confluent.svc.cluster.local:8083 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -E '(Issuer: | DNS:)'

## CHECK THE SERVICE - NO USER
kubectl exec -it kafka-0 -- curl -k https://connect.confluent.svc.cluster.local:8083/

## CHECK THE SERVICE - AUTHENTICATING
kubectl exec -it kafka-0 -- curl -k -u connect-user:connect-password https://connect.confluent.svc.cluster.local:8083/
```

## Test ksqlDB

This is a simple check to confirm the REST endpoint works

```bash
## CHECK GENERATED CERTIFICATE
kubectl exec -it kafka-0 -- \
  openssl s_client -connect ksqldb.confluent.svc.cluster.local:8088 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -E '(Issuer: | DNS:)'

## CHECK THE SERVICE - NO USER
kubectl exec -it kafka-0 -- curl -k https://ksqldb.confluent.svc.cluster.local:8088/info

## CHECK THE SERVICE - NO AUTHENTICATING
kubectl exec -it kafka-0 -- curl -k -u ksqldb-user:ksqldb-password https://ksqldb.confluent.svc.cluster.local:8088/info
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

## Open your browser and navigate to https://localhost:9021
```

**NOTE:** *When using a self-signed certificates, your browser will display a `NET::ERR_CERT_AUTHORITY_INVALID` error message, dependening on the browser there are mechanisms to override and accept the risk of insecure browsing and proceed to C3 page, optionally, you can import the CA cert in your SO/browser certificate trust chain, and restart the browser.

If you have not tear down the producer application, you should see the topic and its content.

**IMPORTANT:** Now you will be prompted to input the user credentials to login into C3, use `admin-user` / `admin-password` or any valid used in the `basic-users-c3` secret created before.

### Test C3/SR/ksqlDB

Use the following queries to test Schema Registry and ksqldb from within C3

```sql
CREATE STREAM users  (id INTEGER KEY, gender STRING, name STRING, age INTEGER) WITH (kafka_topic='users', partitions=1, value_format='AVRO');
```

```sql
INSERT INTO users (id, gender, name, age) VALUES (0, 'female', 'sarah', 42);
INSERT INTO users (id, gender, name, age) VALUES (1, 'male', 'john', 28);
INSERT INTO users (id, gender, name, age) VALUES (42, 'female', 'jessica', 70);
```

**NOTE:** Push Queries will seem "hanged" and will not work since push queries need to open a web socket connection from ksqldb from your browser, and ksqldb is not reachable from the outside in this exercise, the query remains just for completness.

But you can still check the `users` topic content and it's assigned schema in the `Topics` sections on the left side of C3 UI.

```sql
-- Make sure to set auto.offset.reset=earliest
SELECT id, gender, name, age FROM users WHERE age<65 EMIT CHANGES;
-- You should get 2 records
```

## Tear down the platform

```bash
kubectl delete -f cp-platform.yaml

kubectl delete -f secrets/secrets.yaml
```
