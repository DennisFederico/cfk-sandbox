# CFK-SANDBOX
Confluent for Kubernetes deployment with TLS user-provided certificates, RBAC backed by LDAP and with different auth configurations, LDAP, SASL-PLAIN and mTLS.
Kafka Listeners:
* INTERNAL/REPLICATION -> mTLS
* EXTERNAL -> LDAP
* ADHOC (Custom) -> SASL/PLAIN

## Requirements
K8s cluster accessible via kubectl is required.
Helm needed to install CFK

## LDAP for RBAC
MDS is needed for RBAC and it's usually configured with [LDAP](ldap/README.md) as user/groups provider.

## TLS
In order to properly encrypt communication between componenets Server/Client certificates must be provided.
See... [tls](tls/README.md)

## Authentication Credentials and Secrets
Credential to connect from the component services to MDS and the credentias for the ADHOC Kafka Listener.
See... [creds](creds/README.md)

## Kubernetes Secrets
Creation of the kubernetes secrets that holds credentials and certificates, commands are included in the "creds" section above
See... [creds](creds/README.md)

## Deploy the platform
Kubectl apply provided YAMLs per service/component or you can use the `cluster.yaml` that contains them all.

```bash
$ kubectl apply -f cluster.yaml
```

## Sanity Checks
TODO

## TEAR DOWN
Removing the services using the bundled `cluster.yaml` descriptor may leave some Confluent Rolebinding in a 'DELETE-REQUESTED' state as the KafkaRestClass used by the CFK operator are deleted before the RBs.
To manually remove them in such cases, refer to - https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/troubleshooting

Thus it is preferred to remove each component independently one by one using each yaml file, even when the bundled descriptor was used for install.
Any other Confluent Rolebinding should be deleted before kafka and zookeeper ... C3 > KSQLDB > CONNECT > SR > ADHOC RBs > KAFKA > ZK