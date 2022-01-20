There are two choices for installing LDAP

### LDAP in K8s
Helm chart is provided in the [openldap](openldap) directory.
The `values.yaml` is used to configure de domain, access credentials, groups, users, etc
`ldaps-rbac.yaml` is used for TLS when tls.enabled=true in `values.yaml`

```bash install
# Install - helm upgrade --install <relName> --namespace <namespace> [-f <additional descriptor files>] pathToChart
$ helm upgrade --install openldap --namespace confluent -f openldap/ldaps-rbac.yaml ./openldap

# Uninstall - helm uninstall <relName>
$ helm unistall openldap
```

TODO: phpLdapAdmin pod (will need external LB)


### LDAP as VM
This method requires a small VM on the cloud provider reachable by the K8s cluster, either using the same VPC or VPC-Peering
Assuming Debian or Ubuntu

See... [ldpaOnVM.md](ldapOnVM.md)
