# Deploy OpenLdap

## Create a namespace for this service

```bash
kubectl create namespace ldap
```

## Deploy Helm chart with

The chart is configured using the `values.yaml` file

```bash
helm upgrade --install \
  open-ldap openldap \
  -f openldap/ldaps-rbac.yaml \
  --namespace ldap
```

## Test the services

Need to run `ldapsearch` inside the Kubernetes cluster

```bash
kubectl exec -it ldap-0 -n ldap -- bash

ldapsearch -LLL -x -H ldap://ldap.ldap.svc.cluster.local:389 -b 'dc=confluent,dc=acme,dc=com' -D "cn=mds,dc=confluent,dc=acme,dc=com" -w 'Developer!'

```

## Changing LDAP content

In the way it is provided, there's no interface like (i.e. [phpLDAPAdmin](http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page) ) to manage LDAP users, thus you will need to edit the [values.yaml](openldap/values.yaml) file to create/modify users and group, as well as changing the LDAP Base CN.

There's a caveat to applying changes this way, you need to `unistall` the chart but the volumes are persistent and need to be manually deleted before re-installing the chart, see below).

## Remove the Service

```bash
helm uninstall open-ldap -n ldap
kubectl delete pvc ldap-data-ldap-0 -n ldap
kubectl delete pvc ldap-config-ldap-0 -n ldap
```
