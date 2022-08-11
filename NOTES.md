### USERS FOR KAFKA
Create a file with a similar json content (assume `kafka-plain-users.json`)

```
{
"client1": "client1-secret",
"app1": "app1-secret",
"admin": "admin-secret"
}
```

Create a secret for the file
```bash
kubectl create secret generic kafka-users --from-file=plain-users.json=kafka-plain-users.json 
```


#### TIPS TO ENCODE THE SECRETS IN A FILE...

##### Create file from scratch
```bash
kubectl create secret generic kafka-users --from-file=plain-users.json=kafka-plain-users.json --dry-run=client --output=yaml > secret.yaml
```

##### REPLACE VALUE

Given a base `secrets.yaml.tmpl` file

```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kafka-users
  namespace: confluent
data:
  plain-users.json: PLAIN_USERS_JSON
```

```bash
sed "s/PLAIN_USERS_JSON/`cat kafka-plain-users.json|base64 -w0`/g" secrets.yml.tmpl | \
sed <chain the next replacement> | \
 > secrets.yaml
```