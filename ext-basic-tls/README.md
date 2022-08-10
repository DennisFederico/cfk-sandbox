# CREATE SECRET

```bash
echo '---' > secrets.yaml

kubectl create secret generic basic-users-sr \
 --from-file=basic.txt=basic-users-sr-withrole.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-users-connect \
 --from-file=basic.txt=basic-users-connect.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-users-ksqldb \
 --from-file=basic.txt=basic-users-ksqldb-withrole.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-users-c3 \
 --from-file=basic.txt=basic-users-c3-withrole.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-c3-sr-secret \
 --from-file=basic.txt=basic-c3-sr-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-c3-connect-secret \
 --from-file=basic.txt=basic-c3-connect-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-c3-ksqldb-secret \
 --from-file=basic.txt=basic-c3-ksqldb-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-c3-kafka-secret \
 --from-file=plain.txt=basic-c3-kafka-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic kafka-internal-users-jaas \
 --from-file=plain-users.json=kafka-internal-users.json \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic kafka-external-users-jaas \
 --from-file=plain-users.json=kafka-external-users.json \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-mr-kafka-secret \
 --from-file=plain.txt=basic-mr-kafka-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-sr-kafka-secret \
 --from-file=plain.txt=basic-sr-kafka-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-connect-kafka-secret \
 --from-file=plain.txt=basic-connect-kafka-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic basic-ksqldb-kafka-secret \
 --from-file=plain.txt=basic-ksqldb-kafka-secret.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml
```

## Additional EXTERNA TLS secret
```
kubectl create secret generic external-tls \
  --from-file=fullchain.pem=../tlscerts/generated/externalServer.pem \
  --from-file=cacerts.pem=../tlscerts/generated/ExternalCAcert.pem \
  --from-file=privkey.pem=../tlscerts/generated/externalServer-key.pem
```


# Add the Kubernetes NginX Helm repo and update the repo
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx -n confluent
```

# Install the Nginx controller
```bash
helm upgrade  --install ingress-nginx ingress-nginx/ingress-nginx -n confluent
```

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls

```bash
kubectl create secret tls ingress-tls \
  --cert=../tlscerts/generated/externalServer.pem \
  --key=../tlscerts/generated/externalServer-key.pem \
  --namespace confluent
```

# CHECK External TLS (KAFKA)

Find external IP and add to the DNS 
`k get svc kafka-bootstrap-lb -o jsonpath='{.status.loadBalancer.ingress[*].ip}'`
Add each broker to the DNS too
```bash
BOOTSTRAP_IP=$(k get svc kafka-bootstrap-lb -o jsonpath='{.status.loadBalancer.ingress[*].ip}')

$ openssl s_client -connect $BOOTSTRAP_IP:9092 </dev/null 2>/dev/null | openssl x509 -noout -text | grep Issuer:

$ openssl s_client -connect $BOOTSTRAP_IP:9092 </dev/null 2>/dev/null | openssl x509 -noout -text | grep DNS:
```

# CHECK External TLS (SR)

Find external IP and add to the DNS 
`k get svc schemaregistry-bootstrap-lb -o jsonpath='{.status.loadBalancer.ingress[*].ip}'`
Add each broker to the DNS too
```bash
BOOTSTRAP_IP=$(k get svc schemaregistry-bootstrap-lb -o jsonpath='{.status.loadBalancer.ingress[*].ip}')

$ openssl s_client -connect $BOOTSTRAP_IP:443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep Issuer:

$ openssl s_client -connect $BOOTSTRAP_IP:443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep DNS:
```



# TEST SCHEMA REGISTRY

```bash
kubectl exec -it kafka-0 -- curl -k -u c3-user:c3-secret https://schemaregistry.confluent.svc.cluster.local:8081/schemas/types
```

# TEST CONNECT
```bash
kubectl exec -it kafka-0 -- curl -k -u c3-user:c3-secret https://connect.confluent.svc.cluster.local:8083/
```

# TEST KSQLDB
```bash
kubectl exec -it kafka-0 -- curl -k -u c3-user:c3-secret https://ksqldb.confluent.svc.cluster.local:8088/info
```

