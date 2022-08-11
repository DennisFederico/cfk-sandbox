# TLS Certificates

This folders provide an opinionated way to create self-signed certificates that can be used for the exercises that require them.

## Requirements

- Openssl
- [cfssl](https://cfssl.org/) (CloudFlare's SSL toolkit)

```bash
sudo apt install golang-cfssl 
```

## Prepare INTERNAL Certificates

### Self-signing Internal CA Cert and Key

Pre-create a Authorities one for the K8s Cluster (Internal).

```bash
$ openssl genrsa -out generated/InternalCAkey.pem 2048

$ openssl req -x509 -new -nodes \
  -key generated/InternalCAkey.pem \
  -days 365 \
  -out generated/InternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=InternalCA"
```

### Internal (to K8s) certificates

We will rely on [CFK Certificate autogeneration](https://docs.confluent.io/operator/current/co-network-encryption.html#auto-generated-tls-certificates). To make this work we need to provide the signing CA Cert and Key is a secret for CFK, by default this secret should be called `ca-pair-sslcerts` of Type `tls`

It is advisable to create a Secret RD to manage this secret

```bash
kubectl create secret tls ca-pair-sslcerts \
--cert=generated/InternalCAcert.pem \
--key=generated/InternalCAkey.pem \
-n confluent --dry-run=client -output yaml > ca-pair-sslcerts.yaml

kubectl apply -f ca-pair-sslcerts.yaml
```

## Prepare EXTERNAL Certificates

These certificates are used by the exercise (ext-*).

### Self-signing External CA Cert and Key

Generate one CA certificate and key to sign "external" to K8s cluster certificates.

```bash
$ openssl genrsa -out generated/ExternalCAkey.pem 2048

$ openssl req -x509 -new -nodes \
  -key generated/ExternalCAkey.pem \
  -days 365 \
  -out generated/ExternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=ExternalCA"
```

### Create CFSSL Profiles

A simple file with the usages for a certificate profile `cert-req-conf.json`

```json
{
    "signing": {
        "default": {
            "expiry": "1440h"
        },
        "profiles": {
            "server": {
                "expiry": "1440h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "client": {
                "expiry": "1440h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}
```

### Create Server Certificate Request

For make things easier we will use a single certificate for Kafka and one for the rest of the services

Create a defining the CN and [SANs](https://docs.confluent.io/operator/current/co-network-encryption.html#define-san)s for the certificate

Example `external-kafka-cert-req.json`

```json
{
  "CN": "external.kafka.confluent.acme.com",
  "hosts": [
    "bootstrap.kafka.confluent.acme.com",
    "broker-0.kafka.confluent.acme.com",
    "broker-1.kafka.confluent.acme.com",
    "broker-2.kafka.confluent.acme.com",
    "*.kafka.confluent.acme.com"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "Universe",
      "ST": "Pangea",
      "L": "Earth"
    }
  ]
}
```

Example `external-services-cert-req.json`

```json
{
  "CN": "external.services.confluent.acme.com",
  "hosts": [
    "schemaregistry.confluent.acme.com",
    "*.schemaregistry.confluent.acme.com",
    "connect.confluent.acme.com",
    "*.connect.confluent.acme.com",
    "ksqldb.confluent.acme.com",
    "*.ksqldb.confluent.acme.com",
    "controlcenter.confluent.acme.com"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "Universe",
      "ST": "Pangea",
      "L": "Earth"
    }
  ]
}
```

### Generate the EXTERNAL Certificate and Key for Kafka and Services

```bash
$ cfssl gencert -ca=generated/ExternalCAcert.pem \
    -ca-key=generated/ExternalCAkey.pem \
    -config=cert-req-conf.json \
    -profile=server external-kafka-cert-req.json \
    | cfssljson -bare generated/externalKafka

$ cfssl gencert -ca=generated/ExternalCAcert.pem \
    -ca-key=generated/ExternalCAkey.pem \
    -config=cert-req-conf.json \
    -profile=server external-services-cert-req.json \
    | cfssljson -bare generated/externalServices
```

### Generate Secrets to hold the certificates

#### Secret for Kafka external access

```bash
kubectl create secret generic kafka-external-tls \
  --from-file=fullchain.pem=generated/externalKafka.pem \
  --from-file=privkey.pem=generated/externalKafka-key.pem \
  --from-file=cacerts.pem=generated/ExternalCAcert.pem \
  --namespace confluent
```

Note the time of secret difference since this is going to be used by NGINX Ingress Controller

```bash
kubectl create secret tls services-external-tls \
  --cert=generated/externalServices.pem \
  --key=generated/externalServices-key.pem \
  --namespace confluent
```

## MDS Token signing Cert/Key

```bash
openssl genrsa -out mds-priv-key.pem 2048
openssl rsa -in mds-priv-key.pem -out PEM -pubout -out mds-pub-key.pem
```
