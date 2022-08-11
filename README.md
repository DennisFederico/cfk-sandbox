# CFK-SANDBOX

This repository comprise a few examples of simple deployments using CKF.

- [notls-noauth](notls-noauth/) - Most basic setup without Authentication and no TLS (Encryption)
- [tls-noauth](tls-noauth/) - No Authentication, "internal" TLS encryption (self-signed & auto genereated certs)
- [tls-basic](tls-basic/) - Basic/Plain (username & password) Authentication for all services, using "internal" TLS encryption (self-signed & auto generated certs)
- [ext-basic-tls](ext-basic-tls/) - Basic/Plain (username & password) Authentication, "internal" TLS encryption, "external" provided certs via Ingress (SR, Connect, ksqldb & C3) and LB for Kafka
- [ext-rbac-tls](ext-rbac-tls/) - RABC Authorization and "external" provided certs, Includes an "embedded" LDAP deployment
  
## Requirements

- [Helm](https://helm.sh/docs/intro/install/) - to install confluentinc operator and nginx ingress charts
- [kubectl](https://kubernetes.io/docs/tasks/tools/) - to operate you deployments
- Kubernetes Cluster (and kubectl configured to managed such cluster)

## CA and Server Certificates preparation

See to [tlscerts](tlscerts/) folder

## Namespace

For these demos we will be using `confluent` namespace. To segregate your deployment in multiple namespaces, you need to deploy a [customized CFK operator](https://docs.confluent.io/operator/current/co-deploy-cfk.html#configure-co-to-manage-cp-components-across-all-namespaces) to manage Confluent Platform across several namespaces. Additionally for the examples that use nginx Ingress, this need to be deployed on each namespace that requires it.

```bash
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
```

## Deploy CFK via HELM

[Confluent for Kubernetes](https://docs.confluent.io/operator/current/co-deploy-cfk.html) (CFK) deployed from a Helm chart, asuming v.+2.4.0
```bash
helm repo add confluentinc https://packages.confluent.io/helm
helm repo add confluentinc https://packages.confluent.io/helm
```

## Deploy NGINX Ingress

You will need to set up and Ingress controller, for some exercises (ext-basic-tls and ext-rbac-tls) that require TLS termination of the request for HTTPS based services, and re-encrypt the response with an "external" facing certificate.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx -n confluent
```