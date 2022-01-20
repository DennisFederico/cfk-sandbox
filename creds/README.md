## TLS Secrets
TLS Certificates for each component needs to be stored as Kubernetes Secrets for convenience when using CFK

```
## ASSUMING THESE COMMANDS ARE RAN FROM THIS FOLDER

kubectl create secret generic tls-zookeeper \
  --from-file=fullchain.pem=../tls/generated/zookeeper-server.pem \
  --from-file=cacerts.pem=../tls/generated/cacerts.pem \
  --from-file=privkey.pem=../tls/generated/zookeeper-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-kafka \
  --from-file=fullchain.pem=../tls/generated/kafka-server.pem \
  --from-file=cacerts.pem=../tls/generated/cacerts.pem \
  --from-file=privkey.pem=../tls/generated/kafka-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-c3 \
  --from-file=fullchain.pem=../tls/generated/controlcenter-server.pem \
  --from-file=cacerts.pem=../tls/generated/cacerts.pem \
  --from-file=privkey.pem=../tls/generated/controlcenter-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-sr \
  --from-file=fullchain.pem=../tls/generated/schemaregistry-server.pem \
  --from-file=cacerts.pem=../tls/generated/cacerts.pem \
  --from-file=privkey.pem=../tls/generated/schemaregistry-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-connect \
  --from-file=fullchain.pem=../tls/generated/connect-server.pem \
  --from-file=cacerts.pem=../tls/generated/cacerts.pem \
  --from-file=privkey.pem=../tls/generated/connect-server-key.pem \
  --namespace confluent

kubectl create secret generic tls-ksqldb \
  --from-file=fullchain.pem=../tls/generated/ksqldb-server.pem \
  --from-file=cacerts.pem=../tls/generated/cacerts.pem \
  --from-file=privkey.pem=../tls/generated/ksqldb-server-key.pem \
  --namespace confluent
```

## MDS TOKEN KEY PAIR
Provided as `mds-token-public-pair.pem` and `mds-token-public-pair.pem`. These were generated using the following `openssl` commands:
```bash
$ openssl genrsa -out mds-token-private-pair.pem 2048
$ openssl rsa -in mds-token-private-pair.pem -out PEM -pubout -out  mds-toke-public-pair.pem
```

We need this pair (private/public) for MDS to generate tokens

Store the pair in a Kubernetes secret
```bash
kubectl create secret generic mds-token \
	--from-file=mdsPublicKey.pem=mds-token-public-pair.pem \
	--from-file=mdsTokenKeyPair.pem=mds-token-private-pair.pem \
	--namespace confluent
```

## MDS - LDAP Credentials
MDS Needs the credentials to connect with LDAP and authenticate users, as well as fetch the list of groups and users.
Example provided as `ldap.txt` with the following content:

```text
username=cn=mds,dc=dfederico-confluent,dc=com
password=mds-secret
```
```bash
$ kubectl create secret generic ldap-credentials \
	--from-file=ldap.txt=ldap.txt \
	--namespace confluent
```

## MDS Credentials (RBAC)
Each service/component that connects to MDS for authentication and authorization needs its user/password set in a secret
These files contain the username/password for each service

```text
# kafka-mds-client.txt
username=kafka
password=kafka-secret

# sr-mds-client.txt
username=sr
password=sr-secret

# connect-mds-client.txt
username=connect
password=connect-secret

# ksql-mds-client.txt
username=ksql
password=ksql-secret

# c3-mds-client.txt
username=c3
password=c3-secret
```

And for these ofcourse we create secrets (Note the last one for RestClass uses the same credentials as kafka)

```bash
# Kafka RBAC credential
kubectl create secret generic kafka-mds-client \
  --from-file=bearer.txt=kafka-mds-client.txt \
  --namespace confluent

# Schema Registry RBAC credential
kubectl create secret generic sr-mds-client \
  --from-file=bearer.txt=sr-mds-client.txt \
  --namespace confluent

# Connect RBAC credential
kubectl create secret generic connect-mds-client \
  --from-file=bearer.txt=connect-mds-client.txt \
  --namespace confluent

# ksqlDB RBAC credential
kubectl create secret generic ksqldb-mds-client \
  --from-file=bearer.txt=ksqldb-mds-client.txt \
  --namespace confluent

# Control Center RBAC credential
kubectl create secret generic c3-mds-client \
  --from-file=bearer.txt=c3-mds-client.txt \
  --namespace confluent

# Kafka REST credential
kubectl create secret generic kafkarestclass-mds-client \
  --from-file=bearer.txt=kafka-mds-client.txt \
  --from-file=basic.txt=kafka-mds-client.txt \
  --namespace confluent
```

## External Listener SASL-PLAIN Login Module Passthrough
The authentication configuration for External listener is also provided as a secret, 
an for it to authenticate against LDAP (MDS) we provide a jaas configuration with the sasl login module without
credentials as they will be "passed through".

```properties
# kafka-external-sasl-passthrough.jaas
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required;
```

```bash
$ kubectl create secret generic kafka-external-listener-jaas \
    --from-file=plain-jaas.conf=kafka-external-sasl-passthrough.jaas \
    --namespace confluent
```

## CUSTOM Plain Listener
This is an additional kafka listener only added as an example of the authentication alternative to mTLS used in the Internal and the LDAP used for external.
For this we need the list of authenticable users configured on the listener.

Usually provided as a json file (i.e. kafka-adhoc-users.json)
```json
{
  "kafka_client": "kafka_client-secret",
  "devuser": "dev-password",
  "dfederico": "dennis-secret",  
  "james": "james-secret",
  "c3": "c3-secret"
}
```

Added as a secret in kubernetes
```bash
kubectl create secret generic kafka-adhoc-users \
    --from-file=plain-users.json=kafka-adhoc-users.json \
    --namespace confluent
```