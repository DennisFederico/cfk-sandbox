# Secrets for this exercise

## MDS Token Key Pair

```bash
mkdir generated

openssl genrsa -out generated/mds-key-priv.pem 2048
openssl rsa -in generated/mds-key-priv.pem -out PEM -pubout -out generated/mds-key-pub.pem
```

These components are added in the secrets.yml file below


## Create a secrets.yaml file

This series of command can be used the create the secret.yaml file.

```bash
echo '---' > secrets.yaml

kubectl create secret generic kafka-interbroker-creds \
 --from-file=plain-interbroker.txt=kafka-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic mds-key-pair \
 --from-file=mdsPublicKey.pem=generated/mds-key-pub.pem \
 --from-file=mdsTokenKeyPair.pem=generated/mds-key-priv.pem \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic mds-token \
 --from-file=mdsPublicKey.pem=generated/mds-key-pub.pem \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic mds-ldap-creds \
 --from-file=ldap.txt=mds-ldap-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic kafka-mds-creds \
 --from-file=bearer.txt=kafka-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic sr-mds-creds \
 --from-file=bearer.txt=sr-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic connect-mds-creds \
 --from-file=bearer.txt=connect-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic ksql-mds-creds \
 --from-file=bearer.txt=ksql-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic connect-sr-basic \
 --from-file=basic.txt=connect-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml

echo '---' >> secrets.yaml

kubectl create secret generic ksql-sr-basic \
 --from-file=basic.txt=ksql-user-creds.txt \
 --namespace confluent \
 --dry-run=client \
 --output yaml >> secrets.yaml


```

Apply the secrets file

```bash
kubectl apply -f secrets.yaml
```
