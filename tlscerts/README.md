# Self-signed Certificates
This were prepared using cfssl (Cloud Flare SSL) tools, a wrapper over openssl to simplify its usage, thus `openssl`must be installed too.

## Basic Preparation and CA Certificates

### Create "generated" directory
```bash
$ mkdir generated
```

### Install CFSSL
```bash
$ sudo apt install golang-cfssl 
```

### Create CFSSL Profiles
A simple file with the usages for a certificate profile
-- cert-req-conf.json
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

### INTERNAL and EXTERNAL CA Key
```bash
$ openssl genrsa -out generated/InternalCAkey.pem 2048
$ openssl genrsa -out generated/ExternalCAkey.pem 2048
```

### INTERNAL and EXTERNAL CA Certificate
```bash
$ openssl req -x509 -new -nodes \
  -key generated/InternalCAkey.pem \
  -days 365 \
  -out generated/InternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=InternalCA"

$ openssl req -x509 -new -nodes \
  -key generated/ExternalCAkey.pem \
  -days 365 \
  -out generated/ExternalCAcert.pem \
  -subj "/C=ES/ST=VLC/L=VLC/O=Demo/OU=GCP/CN=ExternalCA"
```

## (Option 1) Auto-Generated INTERNAL Certificates
[Documentation](https://docs.confluent.io/operator/current/co-network-encryption.html#auto-generated-tls-certificates)

```bash
$ kubectl create secret tls internal-ca \
  --cert generated/InternalCAcert.pem \
  --key generated/InternalCAkey.pem
```

## (Option 2) User-Provided INTERNAL Servers Certificates
Take into account the [SAN naming](https://docs.confluent.io/operator/current/co-network-encryption.html#define-san) for provided certificates

### Create Server Request
A file with the CN and [SAN](https://docs.confluent.io/operator/current/co-network-encryption.html#define-san)s for the certificate
```json
{
  "CN": "internal.confluent.svc.cluster.local",
  "hosts": [
    "zookeeper.confluent.svc.cluster.local",
    "zookeeper-0.confluent.svc.cluster.local",
    "kafka.confluent.svc.cluster.local",
    "kafka-0.confluent.svc.cluster.local",
    "kafka-1.confluent.svc.cluster.local",
    "kafka-2.confluent.svc.cluster.local",
    "schemaregistry.confluent.svc.cluster.local",
    "schemaregistry-0.confluent.svc.cluster.local",
    "connect.confluent.svc.cluster.local",
    "connect-0.confluent.svc.cluster.local",
    "connect-1.confluent.svc.cluster.local",
    "ksqldb.confluent.svc.cluster.local",
    "ksqldb-0.confluent.svc.cluster.local",
    "controlcenter.confluent.svc.cluster.local",
    "controlcenter-0.confluent.svc.cluster.local"
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

### Generate the INTERNAL Server Certificate
```bash
$ cfssl gencert -ca=generated/InternalCAcert.pem \
    -ca-key=generated/InternalCAkey.pem \
    -config=cert-req-conf.json \
    -profile=server internal-cert-req.json \
    | cfssljson -bare generated/internalServer
```

## User-Provided EXTERNAL Servers Certificates

### Create Server Request
A file with the CN and [SAN](https://docs.confluent.io/operator/current/co-network-encryption.html#define-san)s for the certificate "external"


```json
{
  "CN": "external.confluent.acme.com",
  "hosts": [
    "kafka-bootstrap.confluent.acme.com",
    "kafka-0.confluent.acme.com",
    "kafka-1.confluent.acme.com",
    "kafka-2.confluent.acme.com",
    "schemaregistry.confluent.acme.com",
    "connect.confluent.acme.com",
    "ksqldb.confluent.acme.com",
    "controlcenter.confluent.acme.com",
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

### Generate the INTERNAL Server Certificate
```bash
$ cfssl gencert -ca=generated/ExternalCAcert.pem \
    -ca-key=generated/ExternalCAkey.pem \
    -config=cert-req-conf.json \
    -profile=server external-cert-req.json \
    | cfssljson -bare generated/externalServer
```


## MDS Credentials
```
```bash
### We need to provide a key pair (private/public) for MDS to generate tokens
$ openssl genrsa -out mds-priv-key.pem 2048
$ openssl rsa -in mds-priv-key.pem -out PEM -pubout -out mds-pub-key.pem
```
