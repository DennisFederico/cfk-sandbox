# WORK IN PROGRESS -> TLS with External Re-Encrypt and RBAC

This exercise adds [`RBAC`](https://docs.confluent.io/operator/current/co-rbac.html) for authorization, leveraging on [LDAP Authentication](https://docs.confluent.io/operator/current/co-authenticate.html#co-authenticate-kafka-plain-ldap). It contains several changes from the previous exercise [ext-basic-tls](../ext-basic-tls/), starting with the inclusion of [`MDS`](https://docs.confluent.io/platform/current/kafka/configure-mds/overview.html) service to drive RBAC authorization mechanism.

This exercise will also use the Ingress Controller to access Kafka from the outside of the k8s cluster, this way there's only a single DNS record to maintain.

---

## Spin-up LDAP

To follow this exercise you can deploy an LDAP service in another namespace of your cluster with the provided HELM chart in [ldap-service](ldap-service/README.md) folder.

---

## MDS

This is configured as a service in the Kafka CRD.

```yaml
spec:
  ...
  services:
    mds:
      tls:
        enabled: true
      tokenKeyPair:
        secretRef: mds-key-pair
      provider:
        type: ldap
        ldap:
          address: ldap://ldap.ldap.svc.cluster.local:389
          authentication:
            type: simple
            simple:
              secretRef: mds-ldap-creds
          configurations:
            groupNameAttribute: cn
            groupObjectClass: group
            groupMemberAttribute: member
            groupMemberAttributePattern: CN=(.*),DC=confluent,DC=acme,DC=com
            groupSearchBase: dc=confluent,dc=acme,dc=com
            userNameAttribute: cn
            userMemberOfAttributePattern: CN=(.*),DC=confluent,DC=acme,DC=com
            userObjectClass: organizationalRole
            userSearchBase: dc=confluent,dc=acme,dc=com
          # Additional TLS Configuration available to connect with LDAP
          # tls:
  ...
```

**NOTE:** With also need to add KafkaRest as an actual dependency in the Kafka CRD, to enable/configure the broker admin REST api, with credentials to connect with mds.

```yaml
spec:
  ...
  dependencies:
    ...
    kafkaRest:
      authentication:
        type: bearer
        bearer:
          secretRef: kafka-mds-creds
  ...
```

### MDS Token

MDS needs an (asymetric?) key-pair to create authentication TOKENS, bearer of these tokens will use them to authenticate to the services and authenticity will be validated with the public part of the key-pair. Refer to the secrets section [README.md](secrets/README.md).

```bash
mkdir generated

openssl genrsa -out generated/mds-key-priv.pem 2048
openssl rsa -in generated/mds-key-priv.pem -out PEM -pubout -out generated/mds-key-pub.pem

kubectl create secret generic mds-key-pair \
 --from-file=mdsPublicKey.pem=generated/mds-key-pub.pem \
 --from-file=mdsTokenKeyPair.pem=generated/mds-key-priv.pem \
 --namespace confluent
```

### MDS LDAP Credentials

MDS also needs credentiasl to authenticate with LDAP (bind), these must be provided in a secret with a file named `ldap.txt` with username/password values (username as full CN)

```properties
### ldap.txt content example
username=cn=mds,dc=confluent,dc=acme,dc=com
password=Developer!
```

---

## KafkaRestClass

KafkaRestClass is like the "admin" interface that CFK will use for the cluster, and it is used to create topics, rolebindings, schemas, etc., using CRDs, additionally when deploying componentes it will be used to setup some rolebindings too, thus it is important to add it to your deployment file (or just apply it before other components but after Kafka).

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KafkaRestClass
metadata:
  name: default
  namespace: confluent
  labels:
    component: restclass
spec:
  kafkaClusterRef:
    name: kafka
    namespace: confluent
  kafkaRest:
    authentication:
      type: bearer
      bearer:
        secretRef: kafka-mds-creds
```

---

## Authentication to Kafka from Components

Internal clients and components can keep authenticating using [`SASL/PLAIN`](https://docs.confluent.io/operator/current/co-authenticate.html#co-authenticate-kafka-client-plain).

Components can switch to an `oauthbearer` mechanism to broker a token from MDS, in any case they will need to add MDS as a dependency.

As an example, the `metricReporter` component in Kafka CRD

```yaml
spec:
  ...
  metricReporter:
    enabled: false
    bootstrapEndpoint: kafka:9071
    authentication:
      type: oauthbearer
      oauthbearer:
        secretRef: kafka-mds-creds
    tls:
      enabled: true
  ...
```

Example of component SchemaRegistry CRD authentication with Kafka and MDS Dependency.
**NOTE:** Components connecting to MDS only need the public part of the key-pair

```yaml
spec:
  ...
  dependencies:
    kafka:
      bootstrapEndpoint: kafka:9071
      authentication:
        type: oauthbearer
        oauthbearer:
          secretRef: sr-mds-creds
      tls:
        enabled: true
    mds:
      endpoint: https://kafka.confluent.svc.cluster.local:8090
      tokenKeyPair:
        secretRef: mds-token
      authentication:
        type: bearer
        bearer:
          secretRef: sr-mds-creds
      tls:
        enabled: true
```

## Authentication to Components

Components will now use MDS to check authorization to access their resources, this is accomplished with `spec.authorization.type=rbac` and adding `mds` as a dependency.

Example in SchemaRegistry CRD, in this exercise `mds` is run as a service in the brokers.

```yaml
spec:
  ...
  authorization:
    type: rbac
  ...
  dependencies:
    ...
    mds:
      endpoint: https://kafka.confluent.svc.cluster.local:8090
      tokenKeyPair:
        secretRef: mds-token
      authentication:
        type: bearer
        bearer:
          secretRef: sr-mds-creds
      tls:
        enabled: true
  ...
```

---

## Rolebindings

Rolebindigs should be added by CFK automatically for each component the first time it is deployed, provided the `KafkaRestClass` is deployed.

You can check the rolebindings created using

```bash
kubectl get confluentrolebinding
```

For additional rolebindings you can use a custom CRD like in the following example, that add missing rolebindgs as per our cluster setup, connect user should have access to SR

TODO

---

For each service you can perform the following tests...

## Test External Kafka TLS Cert

Find external IP of any of the broker or the bootstrap, and cCheck the exposed certificates with `openssl`

```bash
BOOTSTRAP_IP=$(kubectl get svc kafka-bootstrap-lb -o jsonpath='{.status.loadBalancer.ingress[*].ip}')

openssl s_client -connect $BOOTSTRAP_IP:9092 </dev/null 2>/dev/null | \
 openssl x509 -noout -text | \
 grep -E '(Issuer: | DNS:)'
```

## Test Kafka Produce and Consume

To consumer and produce to Kafka from outside the K8s cluster, using the external listener, we can use `kafka-topics`, `kafka-producer-perf-test` and `kafka-console-consumer` from the command line.

We first need to setup the properties file to connect and the truststore

```bash
# Import the CA pem file into a jks for trustore
keytool -noprompt -import -alias ca \
  -keystore ../tlscerts/generated/truststore.jks \
  -deststorepass changeme \
  -file ../tlscerts/generated/ExternalCAcert.pem

# Prepare the properties file with the newly created JKS (absolute path)
sed "s|TRUSTORE_LOCATION|$(ls $PWD/../tlscerts/generated/truststore.jks)|g" external-client.properties.tmpl > external-client.properties

```

```bash
## CREATE A TOPIC
kafka-topics --bootstrap-server bootstrap.kafka.confluent.acme.com:9092 \
  --command-config external-client.properties \
  --create \
  --topic app-topic \
  --replication-factor 3 \
  --partitions 1
```

Generate data in one terminal window

```bash
kafka-producer-perf-test \
  --topic app-topic  \
  --record-size 64 \
  --throughput 5 \
  --producer-props bootstrap.servers=bootstrap.kafka.confluent.acme.com:9092 \
  --producer.config external-client.properties \
  --num-records 1000
```

Consume data in another terminal window

```bash
kafka-console-consumer \
  --bootstrap-server bootstrap.kafka.confluent.acme.com:9092 \
  --consumer.config external-client.properties \
  --property print.partition=true \
  --property print.offset=true \
  --topic app-topic \
  --from-beginning \
  --timeout-ms 30000
```






## Testing MDS with confluent CLI

## Configure RBAC

https://docs.confluent.io/operator/current/co-rbac.html




**NOTE:** We could also had use an `externalAccess` of type [`staticForHostBasedRouting`](https://docs.confluent.io/operator/current/co-statichostbased.html#configure-host-based-static-access-to-confluent-components) and route external Kafka access throught the same Ingress controller like the rest of the components.

The change on the Kafka CRD looks like this:

```yaml
spec:
  ...
  listeners:
    ...
    external:
      authentication:
        type: plain
        jaasConfig:
          secretRef: kafka-external-users
      tls:
        enabled: true
        secretRef: external-tls
      externalAccess:
        type: loadBalancer
        loadBalancer:
          domain: confluent.acme.com
          brokerPrefix: kafka
          bootstrapPrefix: kafka-bootstrap
```

**NOTE:** We have also commented the authentication configuration of the ksqldb CRD, and configured the `advertisedUrl` value for the ksqldb dependencie of ***C3*** CDR, the later will allow the browser to make the web socker request (when doing push queries) throught the Ingress. Basic auth for ksqldb from C3 is still new and credentials are not properly sent to ksqldb for push queries.

```yaml
# C3 CRD
spec:
  ...
  dependencies:
    ...
    ksqldb:
      - name: ksqldb        
        url: https://ksqldb.confluent.svc.cluster.local:8088
        advertisedUrl: https://ksqldb.confluent.acme.com
        tls:
          ...          
```

---

## TLS Internal

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

## TLS External

CFK autogenerated certificates SANs for Kafka will always be for the internal DNS of the K8s cluster `<namespace>.svc.cluster.local`, you need to provide your own certificate if you are required to resolve the service to a different DNS, the certificate must contain SANs for the brokers and the bootstrap service, this is also specified when using `loadBalancer` externalAccess.

Refer to the [Prepare EXTERNAL Certificates](../tlscerts/README.md)) in [tlscerts](../tlscerts/) README.md, you should end with two secrets, one holding certificate and key for Kafka external, and another for the certificate and key of the rest of the services.

```bash
kubectl create secret generic kafka-external-tls \
  --from-file=fullchain.pem=../tlscerts/generated/externalKafka.pem \
  --from-file=privkey.pem=../tlscerts/generated/externalKafka-key.pem \
  --from-file=cacerts.pem=../tlscerts/generated/ExternalCAcert.pem
```

```bash
kubectl create secret tls services-external-tls \
  --cert=../tlscerts/generated/externalServices.pem \
  --key=../tlscerts/generated/externalServices-key.pem
```

---

## Service Credentials (Secrets)

For each service we will prepare a secret with a list of allowed users, including "service" user, also secrets with the login/password that each service will use to authenticate among them.
The secrets need are summarized in the following table, additional columns have been provided to help setting up the commands to create them.

This is the same approach from the previous exercise [tls-basic](../tls-basic/README.md), just adding `kafka-external-users` secret.

| Secret | Secret filename | Purpose | Content |
|---|---|---|---|
| kafka-internal-users | plain-users.json | Allowed users to Kafka from internal listener | [file](secrets/kafka-internal-users.json) |
| kafka-external-users | plain-users.json | Allowed users to Kafka from external listener | [file](secrets/kafka-external-users.json) |
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

## Install NGINX Ingress Controller

NGINX Ingress Controller for Kubernetes can be installed using HELM, it must be installed in the same namespace of the services to be exposed.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx -n confluent
```

**NOTE:** Make sure that the chart version is at least `4.2.1` and App version `1.3.0` (using `helm ls` command)

## Create Bootstraps for the Services to expose via the Ingress

Each service with multiple instances would need to be bootstraped for the ingress to redirect to any of them. CFK already creates `ClusterIP` "bootstrap" services for each component with appropiate selectors.

```bash
kubectl describe svc schemaregistry
kubectl describe svc connect
kubectl describe svc ksqldb
```

*NOTE: If by any change the above services are not available or you need a different configuration for these bootstrap that need to be "routed" by the Ingress, see or apply [cp-bootstraps.yaml](cp-bootstraps.yaml)

## Deploy Ingress Rules

Ingress rules example file [cp-ingress-reencrypt.yaml](cp-ingress-reencrypt.yaml) define rules for each service and "dns" based "routing", at the same time, the `tls` definition indicates that TLS should be terminated for the listed domains and "re-encrypted" using the certificate indicated in the `spec.tls.secretName`.

## Update/Add DNS Entries

Once the services are deployed and the IP of the Ingress Controller is available, update your DNS records, you can do this locally via `etc/hosts` file.

To fetch the IP of the Ingress controller...

```bash
INGRESS_IP=kubectl get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[*].ip}'
```

you can use a similar approach for Kakfa or simple check the `EXTERNAL-IP` column when doing `kubectl get svc`

This an example of how the entries in your `etc/hosts` could look like

```text
<INGRESS_IP> schemaregistry.services.confluent.acme.com
<INGRESS_IP> connect.services.confluent.acme.com
<INGRESS_IP> ksqldb.services.confluent.acme.com
<INGRESS_IP> controlcenter.services.confluent.acme.com

<KAFKA_BOOTSTRAP_IP> bootstrap.kafka.confluent.acme.com
<KAFKA_BROKER-0_IP>  broker0.kafka.confluent.acme.com
<KAFKA_BROKER-1_IP>  broker1.kafka.confluent.acme.com
<KAFKA_BROKER-2_IP>  broker2.kafka.confluent.acme.com
```

---



## Test SR (Schema Registry)

This is a simple check to confirm the REST endpoint works

```bash
## CHECK EXTERNAL CERTIFICATE
# Since the ingress works as a reverse proxy, we need an additional argument with the hostname used in the ingress rules... we can connect to the ingress IP instead of the hostname
openssl s_client -connect schemaregistry.services.confluent.acme.com:443 \
  -servername schemaregistry.services.confluent.acme.com </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -E '(Issuer: | DNS:)'

## CHECK THE SERVICE - NO USER
# use -k if you have not added the CA to the trusted chain of yout host
curl -k https://schemaregistry.services.confluent.acme.com/schemas/types

## CHECK THE SERVICE - AUTHENTICATING
# use -k if you have not added the CA to the trusted chain of yout host
curl -k -u sr-user:sr-password https://schemaregistry.services.confluent.acme.com/schemas/types
```

## Test Connect

This is a simple check to confirm the REST endpoint works

```bash
## CHECK EXTERNAL CERTIFICATE
# Since the ingress works as a reverse proxy, we need an additional argument with the hostname used in the ingress rules... we can connect to the ingress IP instead of the hostname
openssl s_client -connect connect.services.confluent.acme.com:443 \
  -servername connect.services.confluent.acme.com </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -E '(Issuer: | DNS:)'

## CHECK THE SERVICE - NO USER
# use -k if you have not added the CA to the trusted chain of yout host
curl -k https://connect.services.confluent.acme.com

## CHECK THE SERVICE - AUTHENTICATING
# use -k if you have not added the CA to the trusted chain of yout host
curl -k -u connect-user:connect-password https://connect.services.confluent.acme.com
```

## Test ksqlDB

This is a simple check to confirm the REST endpoint works

```bash
## CHECK EXTERNAL CERTIFICATE
# Since the ingress works as a reverse proxy, we need an additional argument with the hostname used in the ingress rules... we can connect to the ingress IP instead of the hostname
openssl s_client -connect ksqldb.services.confluent.acme.com:443 \
  -servername ksqldb.services.confluent.acme.com </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -E '(Issuer: | DNS:)'

## CHECK THE SERVICE - NO USER - We have disabled basic auth in this exercise
# use -k if you have not added the CA to the trusted chain of yout host
curl -k https://ksqldb.services.confluent.acme.com/info
```

## Test Confluent Control Center (C3)

Just navigate to [https://controlcenter.services.confluent.acme.com/](https://controlcenter.services.confluent.acme.com/), you will be challenged with a logging, use `admin-user` / `admin-password`

**NOTE:** *When using a self-signed certificates, your browser will display a `NET::ERR_CERT_AUTHORITY_INVALID` error message, dependening on the browser there are mechanisms to override and accept the risk of insecure browsing and proceed to C3 page, optionally, you can import the CA cert in your SO/browser certificate trust chain, and restart the browser.

If you have not tear down the producer application, you should see the topic and its content.

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

**NOTE:** Since we disabled athenticate for ksqldb, the push query should be work properly, as the socket would be created via the ingress (assung `advertisedUrl` for ksql dependency was set in controlcenter CRD)

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


### Authenticating using MDS

### Verify MDS

https://docs.confluent.io/operator/current/co-rbac.html#troubleshooting-verify-mds-configuration