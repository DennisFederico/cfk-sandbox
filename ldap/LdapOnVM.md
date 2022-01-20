https://devconnected.com/how-to-setup-openldap-server-on-debian-10/
https://computingforgeeks.com/install-and-configure-openldap-server-ubuntu/


- Step 1. Intall ***slapd*** and ***ldap-utils***
```bash
$ sudo apt install slapd ldpa-utils
``` 

- Step 2. Configure ***slapd***
```bash
$ sudo dpkg-reconfigure slapd
```
	- Omit OpenLDAP server configuration? No
	- DNS domain name? (i.e. dfederico-confluent.com)
	- Organization name? (i.e. defederico-confluent)
	 - Administrator password? enter a secure password twice
	- Database backend? MDB
	- Remove the database when slapd is purged? No
	- Move old database? Yes
	- Allow LDAPv2 protocol? No

- Step 3. Test Connection
```bash
$ ldapwhoami -H ldap:// -x
$ sudo slapcat
$ ldapsearch -x -LLL -b "" -s base namingContexts
```

- Step 4. (Optional) Install phpldapadmin
https://kifarunix.com/install-phpldapadmin-on-debian-10-debian-11/
```bash
$ wget http://ftp.de.debian.org/debian/pool/main/p/phpldapadmin/phpldapadmin_1.2.2-6.3_all.deb

$ apt install ./phpldapadmin_1.2.2-6.3_all.deb
```
Then configrue the *server* at `/etc/phpldapadmin/config.php`  
Log into http://[serverIp]/phpldapadmin/

---
#### Add a Node using ***ldapadd***
Create a file like this...
```properties
# users-ou.ldif 
dn: ou=Users,dc=dfederico-confluent,dc=com 
objectClass: organizationalUnit
ou: Users
```

Add the node to LDAP
```bash
$ sudo ldapadd -D "cn=admin,dc=dfederico-confluent,dc=com" -W -H ldapi:/// -f users-ou.ldif
```

Quick test
```bash
$ ldapsearch -x -b "dc=dfederico-confluent,dc=com" ou
```

#### Add "service" user
Example creating the user kafka...
```properties
# kafka-user.ldif
dn: cn=kafka,dc=dfederico-confluent,dc=com
userPassword: kafka-secret
description: kafka user
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: kafka
```

Submit the file
```bash
$ sudo ldapadd -D "cn=admin,dc=dfederico-confluent,dc=com" -W -H ldapi:/// -f kafka-user.ldif
```

#### Add "user" to the Users group
Create the ***posixGroups***
```properties
# groups-ou.ldif
dn: ou=Groups,dc=dfederico-confluent,dc=com 
objectClass: organizationalUnit
ou: Groups

# c3users-group.ldif
dn: cn=c3users,ou=Groups,dc=dfederico-confluent,dc=com
objectClass: top
objectClass: posixGroup
cn: c3users
gidNumber: 5000

# readonlyusers-group.ldif
dn: cn=readonlyusers,ou=Groups,dc=dfederico-confluent,dc=com
objectClass: top
objectClass: posixGroup
cn: readonlyusers
gidNumber: 5100

# adminusers-group.ldif
dn: cn=adminusers,ou=Groups,dc=dfederico-confluent,dc=com
objectClass: top
objectClass: posixGroup
cn: adminusers
gidNumber: 5200

```

Create ***posixUsers*** in the Users group
```properties
# users-ou.ldif 
dn: ou=Users,dc=dfederico-confluent,dc=com 
objectClass: organizationalUnit
ou: Users

# dfederico-user.ldif
dn: cn=dennis,ou=Users,dc=dfederico-confluent,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: dfederico
uid: dfederico
givenName: Dennis
sn: Federico
displayName: Dennis Federico
uidNumber: 10009
gidNumber: 5200
userPassword: dennis-secret
homeDirectory: /home/dfederico

# jlogan-user.ldif
dn: cn=jlogan,ou=Users,dc=dfederico-confluent,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: jlogan
uid: jlogan
givenName: James
sn: Logan
displayName: James Logan
uidNumber: 10010
gidNumber: 5100
userPassword: james-secret
homeDirectory: /home/jlogan

# alice-user.ldif
dn: cn=alice,ou=Users,dc=dfederico-confluent,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: alice
uid: alice
givenName: Alice
sn: Cooper
displayName: Alice Cooper
uidNumber: 10011
gidNumber: 9000
userPassword: alice-secret
homeDirectory: /home/alice

```


Add users to other grousp
```properties
# add-users-to-groups
dn: cn=c3users,ou=groups,dc=dfederico-confluent,dc=com
changetype: modify
add: memberUid
memberUid: cn=jlogan,ou=users,dc=dfederico-confluent,dc=com

dn: cn=c3users,ou=Groups,dc=dfederico-confluent,dc=com
changetype: modify
add: memberUid
memberUid: cn=dfederico,ou=users,dc=dfederico-confluent,dc=com

dn: cn=c3users,ou=Groups,dc=dfederico-confluent,dc=com
changetype: modify
add: memberUid
memberUid: cn=alice,ou=users,dc=dfederico-confluent,dc=com

dn: cn=adminusers,ou=Groups,dc=dfederico-confluent,dc=com
changetype: modify
add: memberUid
memberUid: cn=dfederico,ou=users,dc=dfederico-confluent,dc=com

dn: cn=readonlyusers,ou=groups,dc=dfederico-confluent,dc=com
changetype: modify
add: memberUid
memberUid: cn=jlogan,ou=users,dc=dfederico-confluent,dc=com

dn: cn=readonlyusers,ou=groups,dc=dfederico-confluent,dc=com
changetype: modify
add: memberUid
memberUid: cn=alice,ou=users,dc=dfederico-confluent,dc=com
```



---

YOU CAN USE ***sudo slappasswd*** to create SSHA passwords for the userPassword

#### Confluent Platform users
```properties

# cp-users.ldif

dn: cn=kafka,dc=dfederico-confluent,dc=com
userPassword: {SSHA}2jdXG2HgNl7Ci0fQ0F3VVkj/x46d7epY
description: Kafka User
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: kafka

dn: cn=mds,dc=dfederico-confluent,dc=com
userPassword: {SSHA}yDgSkKXC0fmERp58oB+OLVvayw7J5Bu8
description: User for MetaData Service (mds)
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: mds

dn: cn=sr,dc=dfederico-confluent,dc=com
userPassword: {SSHA}Xi3hurbGqa7orQH++j/CSFt3MwmJPLvF
description: Schema Registry User
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: sr

dn: cn=c3,dc=dfederico-confluent,dc=com
userPassword: {SSHA}KENe6CdIUeHNRyLUsGHXSRn5mMA6ccPZ
description: Confluent Control Center User
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: c3

dn: cn=ksql,dc=dfederico-confluent,dc=com
userPassword: {SSHA}HsEqVJbHaDCH3lCn0/HP8PIkC6YEGhqM
description: KsqlDB User
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: ksql

dn: cn=connect,dc=dfederico-confluent,dc=com
userPassword: {SSHA}H1MJo0ALVEXaxodXhh3bJ2HhDMDpmD+5
description: Kafka Connect User
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: connect

dn: cn=replicator,dc=dfederico-confluent,dc=com
userPassword: {SSHA}oHWDmCQxVyIWDKCcWyPVajdiaFArO6sQ
description: Confluent Replicator User
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: replicator
```


---
BONUS... ACL to restrict read access
https://www.openldap.org/doc/admin24/access-control.html

to specif user... ie.
`ldapsearch -xLLL -H ldap://dfederico-openldap.c.solutionsarchitect-01.internal -b "dc=dfederico-confluent,dc=com" -D "cn=ldap-readuser,ou=Users,dc=dfederico-confluent,dc=com" -W ""`


BONUS TLS...
https://computingforgeeks.com/secure-ldap-server-with-ssl-tls-on-ubuntu/
https://ubuntu.com/server/docs/service-ldap-with-tls

```properties
## addcerts.ldif
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/ca_server.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/ldap_server.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/ldap_server.key
```

Apply using...
``` bash
$ldapmodify -H ldapi:// -Y EXTERNAL -f addcerts.ldif
```

Lastly check /etc/default/slapd
```
SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"
```

```bash
### SOME LDAP SEARCHS
$ ldapsearch -LLL -x -H ldap://ldap.dfederico.zone:389 -b 'dc=dfederico-confluent,dc=com' -D 'cn=mds,dc=dfederico-confluent,dc=com' -w 'mds-secret' "(&(objectClass=posixAccount)(ou:dn:=users))" dn



```

**https://tylersguides.com/guides/openldap-memberof-overlay/**https://tylersguides.com/guides/openldap-memberof-overlay/

https://kifarunix.com/how-to-create-openldap-member-groups/

http://blog.oddbit.com/post/2013-07-22-generating-a-membero/

```bash
$ sudo ldapmodify -Y EXTERNAL -H ldapi:/// <<EOL
dn: cn=module{0},cn=config
add: olcModuleLoad
olcModuleLoad: memberof.la
EOL

$ sudo ldapadd -Y EXTERNAL -H ldapi:/// <<EOL
dn: olcOverlay={0}memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
olcOverlay: memberof
EOL

ldapsearch -x -b DC=preproduccion,DC=net -H ldap://TV1IW0001.preproduccion.net:389 -D "CN=srv_ldap_confluent,OU=CONFLUENT PLATFORM,OU=ROLES 2019,DC=preproduccion,DC=net" -w "zx3h5K--P9ap394BZKdhDsYG&&" "cn:=confluent-admin"

```
