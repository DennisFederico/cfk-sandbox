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

