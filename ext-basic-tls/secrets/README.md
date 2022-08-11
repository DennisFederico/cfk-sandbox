# Create a secrets.yaml file

This series of command can be used the create the secret.yaml file.

```bash
echo '---' > secrets.yaml

kubectl create secret generic kafka-internal-users \
 --from-file=plain-users.json=kafka-internal-users.json \
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

echo '---' >> secrets.yaml

kubectl create secret generic basic-ksqldb-sr-secret \
 --from-file=basic.txt=basic-ksqldb-sr-secret.txt \
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
```

Apply the secrets file

```bash
kubectl apply -f secrets.yaml
```
