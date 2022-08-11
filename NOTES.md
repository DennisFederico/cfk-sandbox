# TIP TO REPLACE VALUES IN YAML WITH BASE64 EQUIVALENT

Given a base `secrets.yaml.tmpl` file

```yaml
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
