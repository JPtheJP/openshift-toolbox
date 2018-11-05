# Authentication

These are notes related to authentication.

# LDAP Configuration

First (if using `ldaps`) you need to download the CA certificate (below example is using Red Hat IdM server)

```
root@master# curl  http://ipa.example.com/ipa/config/ca.crt >> /etc/origin/master/my-ldap-ca-bundle.crt
```

Make a backup copy of the config file
```
root@master# cp /etc/origin/master/master-config.yaml{,.bak}
```

Edit the `/etc/origin/master/master-config.yaml` file with the following changes under the `identityProviders` section

```
  identityProviders:
  - name: "my_ldap_provider"
    challenge: true
    login: true
    provider:
      apiVersion: v1
      kind: LDAPPasswordIdentityProvider
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: "cn=directory manager"
      bindPassword: "secret"
      ca: my-ldap-ca-bundle.crt
      insecure: false
      url: "ldaps://ipa.example.com/cn=users,cn=accounts,dc=example,dc=com?uid"
```

Note you can customize what attributes it searches for. First non empty attribute returned is used.

Restart the openshift-master service
```
systemctl restart atomic-openshift-master
```

# Active Directory

AD usually is using `sAMAccountName` as uid for login. Use the following ldapsearch to validate the informaiton

```
ldapsearch -x -D "CN=xxx,OU=Service-Accounts,OU=DCS,DC=homeoffice,DC=example,DC=com" -W -H ldaps://ldaphost.example.com -b "ou=Users,dc=office,dc=example,DC=com" -s sub 'sAMAccountName=user1'
```

If the ldapsearch did not return any user, it means -D or -b may not be correct. Retry different `baseDN`. If there is too many entries returns, add filter to your search. Filter example is `(objectclass=people)` or `(objectclass=person)` if still having issues; increase logging as `OPTIONS=--loglevel=5` in `/etc/sysconfig/atomic-openshift-master`

If you see an error in `journalctl -u atomic-openshift-master`  there might be a conflict with the user identity when user trying to login (if you used `htpasswd` beforehand). Just do the following...
```
oc get user
oc delete user user1
```

Inspiration from [This workshop](https://github.com/RedHatWorkshops/openshiftv3-ops-workshop/blob/master/adding_an_ldap_provider.md)

The configuration in `master-config.yaml` Should look something like this:

```
oauthConfig:
  assetPublicURL: https://master.example.com:8443/console/
  grantConfig:
    method: auto
  identityProviders:
  - name: "OfficeAD"
    challenge: true
    login: true
    provider:
      apiVersion: v1
      kind: LDAPPasswordIdentityProvider
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - sAMAccountName
      bindDN: "CN=LinuxSVC,OU=Service-Accounts,OU=DCS,DC=office,DC=example,DC=com"
      bindPassword: "password"
      ca: ad-ca.pem.crt
      insecure: false
      url: "ldaps://ad-server.example.com:636/CN=Users,DC=hoffice,DC=example,DC=com?sAMAccountName?sub"
```

If you need to look for a subclass...

```
"ldaps://ad.corp.example.com:636/OU=Users,DC=corp,DC=example,DC=com?sAMAccountName?sub?(&(objectClass=person)"
```

Here's an example of doing it inside on an ansible host file

```
```
openshift_master_identity_providers=[{'name':'Active_Directory','login':'true','challenge':'true','kind':'LDAPPasswordIdentityProvider','attributes':{'id':['userPrincipalName'],'email':['userPrincipalName'],'name':['name'],'preferredUsername':['sAMAccountName']},'insecure':'true','bindDN':'CN=svc-openshift,CN=Users,DC=moos,DC=red','bindPassword':'REMOVED','url':'ldap://dc.moos.red:389/CN=Users,DC=moos,DC=red?sAMAccountName?sub?(objectClass=person)'}]
```

# Two Auth Provider

Here is an example of using two auth providers
```
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: htpasswd_auth
    provider:
      apiVersion: v1
      file: /etc/origin/master/htpasswd
      kind: HTPasswdPasswordIdentityProvider
  - challenge: true
    login: true
    mappingMethod: claim
    name: htpasswd_auth2
    provider:
      apiVersion: v1
      file: /etc/origin/master/htpasswd2
      kind: HTPasswdPasswordIdentityProvider
```


The ansible scripts configured authentication using `htpasswd` so just create the users using the proper method

```
root@host# htpasswd -b /etc/origin/openshift-passwd demo demo
```

# Adding User to group

Currently, you can only add a user to a group by setting the "group" array to a group
```
[root@ose3-master ~]# oc edit user/christian -o json
{
    "kind": "User",
    "apiVersion": "v1",
    "metadata": {
        "name": "christian",
        "selfLink": "/osapi/v1beta3/users/christian",
        "uid": "a5c96638-4084-11e5-8a3c-fa163e2e3caf",
        "resourceVersion": "1182",
        "creationTimestamp": "2015-08-11T23:56:56Z"
    },
    "identities": [
        "htpasswd_auth:christian"
    ],
    "groups": [
        "mygroup"
    ]
}

```

# Create Admin User

First, you create the user using either `ldap` or the `htpasswd` file. It's basically whatever backend auth system you set up; for example; if you use `htpasswd` create the user like so...

```
htpasswd /etc/origin/openshift-passwd ocp-admin
```

Then, as `system:admin` give this user `cluster-admin` permissions (**CAREFUL** this is like "root" but for OpenShift)

```
oc adm policy add-cluster-role-to-user cluster-admin ocp-admin
```

Now they can login from anywhere with an `oc` cli tool and login with...

```
user@host$ oc login https://ose3-master.example.com:8443 --insecure-skip-tls-verify --username=ocp-admin
```

You can then maybe install [cockpit](https://github.com/RedHatWorkshops/openshiftv3-ops-workshop/blob/master/deploying_cockpit_as_a_container.md#step-2) if you want a sort of "admin" interface.

# Login

There are many methods to login including username/password or token.

## User Login

To login as a "regular" user...

```
user@host$ oc login https://ose3-master.example.com:8443 --insecure-skip-tls-verify --username=demo
```
## Admin Login

On the master, you can log back into OCP as admin with...

```
root@master# oc login -u system:admin -n default
```

Or, you can specify the kubeconfig file directly

```
 oc login --config=/path/to/admin.kubeconfig -u system:admin
```

You can also export `KUBECONFIG` to wherever your kubeconfig is (when you login, it SHOULD be under `~/.kube/config`  but you can specify the one on the master if you'd like)

# Custom Roles

More info found [here](http://v1.uncontained.io/playbooks/operationalizing/custom_role_creation.html)

Highlevel; find something like what you want and export it.

```
oc export clusterrole edit > edit_role.yaml
cp edit_role.yaml customrole.yaml
```

Edit to your hearts content (I did a `diff` here to show you the change)

```
diff edit_role.yaml customrole.yaml

8c8
<   name: edit
---
>   name: edit_no_rsh
16d15
<   - pods/exec
```

Above you see I changed the name and removed `pods/exec`

You also want to remove the `aggregationRule` (redo the diff because it'll look different than the `diff` above)

```
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-edit: "true"
```

Load the new role
```
oc create -f customrole.yaml
clusterrole "edit_no_rsh" created
```

Assign to a user

```
oc adm policy add-role-to-user edit_no_rsh bob -n myproject
```

