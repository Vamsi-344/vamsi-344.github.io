---
title: Integrating KeyCloak with OpenStack
categories: [OpenStack, KeyCloak]
date: 2024-08-05 23:23
tags: [documentation]
---
Please read about [Keystone Federation]({% post_url 2024-08-05-keystone-federation %})[^1] before moving forward to implementation

Integrating KeyCloak with OpenStack includes four main steps [^2]:

 1. Installation of DevStack in a VM
 2. Install and Configure a KeyCloak instance
 3. Configure Keystone and Shibboleth
 4. Configure Horizon

---
## Installation of DevStack in a VM

After trying to running DevStack [^3] on ubuntu VM, important things to note are:

1. As of now, DevStack attempts to support the two latest LTS releases of Ubuntu, Rocky Linux 9 and openEuler.
2. If you do not have a preference, Ubuntu 22.04 (Jammy) is the most tested, and will probably go the smoothest.
3. After many trails, I was facing errors saying "tenant network" and came across a stackoverflow page [^4], which kinda solved my problem, we need to do some specific network settings in virtual box to overcome this error:
	1. From the Network section, change Attached to to Bridged Adapter, Adapter Type to Paravirtualized Network (virtio-net) and Promiscuous Mode to Allow All.
	2. Check Enable Nested VT-x/AMD-v from System > Processor
		1. We can't directly enable the nesting from the GUI [^5], go to VirtualBox installation folder in windows terminal, in my case `C:\Program Files\Oracle\VirtualBox` and run `VBoxManage modifyvm <YourVirtualMachineName> --nested-hw-virt on`

**Final Decisions:**

1. We will be running ubuntu 22.04 (jammy) server for our devstack
2. We will have two networks, one consisting of bridged adapter for the devstack network with above configuration and another one is host-only adapter for static-ip and accessibility from our local machine

After creating a VirtualBox VM successfully, ssh into the VM from local machine windows terminal using command:
`ssh <username>@<vm-ip>`
and enter the vm password 

My current system configuration:
- IP: 192.168.81.11
- username: groot
- password: iamgroot
### Download DevStack

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
```

### Create a local.conf

```ini
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
HOST_IP=<vm-static-ip-from-host-only-network>
```

### Start the install

```bash
./stack.sh
```

Entry the password if prompted, it takes around 20-30 min for the installation, and we can access the openstack dashboard from the VM ip on our local machine browser

---
## Install and Configure a KeyCloak instance

Create a seperate folder on local machine for preparing the keycloak instance configuration files
and follow the below steps

### Generate self-signed domain cert/key certificates using the commands:

```bash
openssl req -newkey rsa:2048 -nodes \
  -keyout server.key.pem -x509 -days 3650 -out server.crt.pem
```

> openssl can be ran on linux, so run these on the devstack and scp them to the local machine keycloak setup folder
{: .prompt-tip}

### Update permissions for the key

```bash
chmod 755 server.key.pem
```

### Create a docker-compose.yaml

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0.1
    restart: always
    command: start
    environment:
      KC_HTTPS_CERTIFICATE_FILE: /opt/keycloak/conf/server.crt.pem
      KC_HTTPS_CERTIFICATE_KEY_FILE: /opt/keycloak/conf/server.key.pem
      KC_HOSTNAME: auth.localhost
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: password
    ports:
      - "8081:8080"
      - "8443:8443"
    volumes:
      - ${PWD:-.}/server.crt.pem:/opt/keycloak/conf/server.crt.pem
      - ${PWD:-.}/server.key.pem:/opt/keycloak/conf/server.key.pem
```

### Start KeyCloak

```bash
docker compose up -d
```

### Add the keycloak instance host to our VM /etc/hosts

Keycloak instance should be accessible from our DevStack VM, so add the accessible ip into /etc/hosts

```bash
sudo echo "192.168.81.1 auth.localhost" >> /etc/hosts
```

> Do remember that we can access the KeyCloak instance on our DevStack VM from the url [https://auth.localhost:8443](https://auth.localhost:8443), we use 8443 port as we are running https keycloak instance, if http it will be 8080
{: .prompt-info}

### Realm Setup

A single Keycloak server supports multiple realms. Each realm has its own users, groups, and clients. To keep our OpenStack users separate from the rest of our Keycloak instance, we begin by creating a new realm.

You can use any name for your realm, but keep in mind the IdP Entity ID will contain this name. For example if you choose OpenStack as your realm name, the Entity ID will become:

**https://auth.localhost:8443/realms/OpenStack**

You can verify your Entity ID by going to _Realm Settings_ and accessing the _SAML 2.0 Identity Provider Metadata_ under _Endpoints_ in the _General_ settings.

Applications that want to use KeyCloak for authentication must be registered in the system as a client. We, therefore, create a new client for our Keystone instance. Select _Clients_ from the sidebar and click _Create Client_.

The type must be set to SAML, and the Client ID must match the Keystone Entity ID. You can freely choose a (globally) unique Entity ID as long as you later configure Keystone to use that same ID. Here we'll use "**http://192.168.81.11**". The name of client can be chosen freely.

Once created, open the client and make sure the following settings are set:

- Valid redirect URIs  
    **http://192.168.81.11/Shibboleth.sso/SAML2/POST**
- Force nameID Format
- Client Signature Required:
	- off

### Creating a user

Select _Users_ from the sidebar and click _Add user_. The username should match the Keystone username.

---

## Configure Keystone and Shibboleth

### Keystone

The SAML2.0 authentication flow is handled by an Apache module called Shibboleth. Keystone itself just receives the REMOTE_USER parameter with the name of the currently logged in Keycloak user from Shibboleth and has nothing to do with the actual authentication.

### Setup Keystone resources

When a user logs in using Keycloak our the SAML2.0 message contains a unique user identifier. This can be the email address, username, or any other attribute associated with the user in Keycloak. The Shibboleth module extracts this user identifier (see below) and passes it to Keystone through the REMOTE_USER variable.

Because Keystone does not know about Keycloak users, it needs to be told how to map remote (Keycloak) to local (Keystone) identities. Start by creating a temporary file that contains the mapping rules, we'll call it _rules.json_. The mapping below takes the value from REMOTE_USER and finds a local Keystone user with that name.

```json
[
    {
        "local": [
            {
                "user": {
                    "name": "{0}",
                    "type": "local",
                    "domain": {
                        "id": "default"
                    }
                }
            }
        ],
        "remote": [
            {
                "type": "REMOTE_USER"
            }
        ]
    }
]
```

OpenStack Mapping (rules.json)

> Important: we map to existing Keystone users by specifying the type as _local_. If this is omitted a new user is created in the _federated_ domain of Keystone.
{: .prompt-warning}

Now create a new mapping using the OpenStack command line tool. The name of the mapping is arbitrary, here we'll name it _saml_mapping._

```bash
openstack mapping create saml_mapping --rules rules.json
```

Then, create a new identity provider, we'll name it _keycloak_saml_. The remote-id argument should be set to the Keycloak Entity ID.

```bash
openstack identity provider create keycloak_saml --remote-id https://auth.localhost:8443/realms/OpenStack --domain default 
```

Finally create a new federation protocol, we'll name it _saml2,_ using the following command.

```bash
openstack federation protocol create saml2 --mapping saml_mapping --identity-provider keycloak_saml
```

Once that is done, we'll need to change the Keystone configuration to allow SAML2.0 based authentication. Also make sure your Horizon dashboard is listed as a _trusted_dashboard_.

```ini
[auth]
methods = token,saml2,password

[federation]
remote_id_attribute = Shib-Identity-Provider
trusted_dashboard = http://192.168.81.11/dashboard/auth/websso/
sso_callback_template = /etc/keystone/sso_callback_template.html

[saml2]
remote_id_attribute = Shib-Identity-Provider
```

/etc/keystone/keystone.conf

Depending on your distribution, the file sso_callback_template.html might be missing from your /etc/keystone folder. The contents of that file are shown below.

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title>Keystone WebSSO redirect</title>
  </head>
  <body>
     <form id="sso" name="sso" action="$host" method="post">
       Please wait...
       <br/>
       <input type="hidden" name="token" id="token" value="$token"/>
       <noscript>
         <input type="submit" name="submit_no_javascript" id="submit_no_javascript"
            value="If your JavaScript is disabled, please click to continue"/>
       </noscript>
     </form>
     <script type="text/javascript">
       window.onload = function() {
         document.forms['sso'].submit();
       }
     </script>
  </body>
</html>
```

/etc/keystone/sso_callback_template.html

### Shibboleth - Apache

💡

The actual SAML flow is handled by an Apache module. There are two options: Shibboleth and Mellon. We are going to Shibboleth as of now.

Now Keystone knows about our Identity Provider we'll need to configure Apache to use Shibboleth for authentication.

First, install and enable the Apache Shibboleth module.

```bash
apt-get install libapache2-mod-shib
a2enmod shib
```

Install and enable the shib Apache module

Then, change your Apache configuration to require valid authentication when accessing the following endpoints:

```
/identity/v3/OS-FEDERATION/identity_providers/keycloak_saml/protocols/saml2/auth

/identity/v3/auth/OS-FEDERATION/websso/saml2

/identity/v3/auth/OS-FEDERATION/identity_providers/keycloak_saml/protocols/saml2/websso
```

Relevant authentication endpoints

Here, _keycloak_saml_ and _saml2_ refer to the identity provider and federation protocol you've created in the previous step.

```conf
LoadModule mod_shib /usr/lib/apache2/modules/mod_shib.so

ProxyPass "/identity" "unix:/var/run/uwsgi/keystone-wsgi-public.socket|uwsgi://uwsgi-uds-keystone-wsgi-public" retry=0 acquire=1
SSLEngine off
SSLHonorCipherOrder off


WSGIDaemonProcess keystone-public processes=5 threads=1 user=groot group=groot display-name=%{GROUP}
WSGIProcessGroup keystone-public
WSGIScriptAlias / /usr/bin/keystone-wsgi-public
WSGIApplicationGroup %{GLOBAL}
WSGIPassAuthorization On
ErrorLogFormat "%{cu}t %M"
ErrorLog /var/log/apache2/keystone.log
CustomLog /var/log/apache2/keystone_access.log combined

<Directory /usr/bin>
	Require all granted
</Directory>

Alias /identity /usr/bin/keystone-wsgi-public

<Location /identity>
	SetHandler wsgi-script
	Options +ExecCGI

	WSGIProcessGroup keystone-public
	WSGIApplicationGroup %{GLOBAL}
	WSGIPassAuthorization On
</Location>

<Location /Shibboleth.sso>
	SetHandler shib
	Require all granted
</Location>

<Location /identity/v3/OS-FEDERATION/identity_providers/keycloak_saml/protocols/saml2/auth>
	AuthType shibboleth
	Require valid-user
	ShibRequestSetting requireSession 1
	ShibExportAssertion Off
</Location>

<Location /identity/v3/auth/OS-FEDERATION/websso/saml2>
	Require valid-user
	AuthType shibboleth
	ShibRequestSetting requireSession 1
	ShibExportAssertion off
	<IfVersion < 2.4>
		ShibRequireSession On
		ShibRequireAll On
	</IfVersion>
</Location>

<Location /identity/v3/auth/OS-FEDERATION/identity_providers/keycloak_saml/protocols/saml2/websso>
	Require valid-user
	AuthType shibboleth
	ShibRequestSetting requireSession 1
	ShibExportAssertion off
	<IfVersion < 2.4>
		ShibRequireSession On
		ShibRequireAll On
	</IfVersion>
</Location>
```

/etc/apache2/sites-available/keystone-wsgi-public.conf
### Shibboleth

Now Apache is configure to use Shibboleth, we need to configure Shibboleth itself.

We begin by creating a new private key for Shibboleth to use when signing authentication requests:

```bash
cd /etc/shibboleth
shib-keygen -y 1
```

Then, download the Keycloak metadata to a local file:

```bash
wget https://auth.localhost:8443/realms/OpenStack/protocol/saml/descriptor -O /etc/shibboleth/keycloak-metadata.xml
```

Next, we configure Shibboleth by editing /etc/shibboleth/shibboleth2.xml. Make sure the entityID attributes are set correctly and the metadata provider points to the Keycloak SAML2.0 metadata endpoint.

```xml
<SPConfig xmlns="urn:mace:shibboleth:3.0:native:sp:config"
    xmlns:conf="urn:mace:shibboleth:3.0:native:sp:config"
    clockSkew="180">

    <OutOfProcess tranLogFormat="%u|%s|%IDP|%i|%ac|%t|%attr|%n|%b|%E|%S|%SS|%L|%UA|%a" />

    <ApplicationDefaults entityID="http://192.168.81.11/shibboleth" signing="false" encryption="false" REMOTE_USER="nameID" cipherSuites="DEFAULT:!EXP:!LOW:!aNULL:!eNULL:!DES:!IDEA:!SEED:!RC4:!3DES:!kRSA:!SSLv2:!SSLv3:!TLSv1:!TLSv1.1">

            <Sessions lifetime="28800" timeout="3600" relayState="ss:mem" checkAddress="false" handlerURL="/Shibboleth.sso" handlerSSL="false" cookieProps="http">

            <SSO entityID="https://auth.localhost:8443/realms/OpenStack">SAML2</SSO>
            <Logout>SAML2 Local</Logout>
            <LogoutInitiator type="Admin" Location="/Logout/Admin" acl="127.0.0.1 ::1" />
            <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>
            <Handler type="Status" Location="/Status" acl="127.0.0.1 ::1"/>
            <Handler type="Session" Location="/Session" showAttributeValues="false"/>
            <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
        </Sessions>

        <Errors supportContact="root@localhost"
            helpLocation="/about.html"
            styleSheet="/shibboleth-sp/main.css"/>

        <MetadataProvider type="XML" path="/etc/shibboleth/keycloak-metadata.xml">
        </MetadataProvider>

        <!-- Map to extract attributes from SAML assertions. -->
        <AttributeExtractor type="XML" validate="true" reloadChanges="false" path="attribute-map.xml"/>
        <AttributeFilter type="XML" validate="true" path="attribute-policy.xml"/>

        <!-- Simple file-based resolvers for separate signing/encryption keys. -->
        <CredentialResolver type="File" key="sp-key.pem" certificate="sp-cert.pem" />
                <!--<CredentialResolver type="File" use="encryption"
        key="sp-encrypt-key.pem" certificate="sp-encrypt-cert.pem"/>-->

    </ApplicationDefaults>

    <!-- Policies that determine how to process and authenticate runtime messages. -->
    <SecurityPolicyProvider type="XML" validate="true" path="security-policy.xml"/>

    <!-- Low-level configuration about protocols and bindings available for use. -->
    <ProtocolProvider type="XML" validate="true" reloadChanges="false" path="protocols.xml"/>
</SPConfig>
```

/etc/shibboleth/shibboleth2.xml

The REMOTE_USER attribute contains the id of the attribute Shibboleth passes to Keystone as the remote user (here it is called _nameID_). The attribute-map.xml file in the Shibboleth configuration directory determines how these attributes are extracted from the SAML2.0 response. In the example below we use the value Keycloak passes as the nameid.

```xml
<Attributes xmlns="urn:mace:shibboleth:2.0:attribute-map" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <Attribute name="urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified" id="nameID"/>
</Attributes>
```

/etc/shibboleth/attribute-map.xml

> Note that nameid is a default SAML2.0 attribute. The unspecified format indicates that the value provided is application defined, in our case it contains the Keycloak username.
{: .prompt-danger}

## Configure Horizon

Now both Keycloak and Keystone are properly configured, we need to let Horizon know that the SAML2.0 authentication flow is enabled. The parts of the configuration that are relevant to our setup are listed below.

```python
# Set this to true to enable debug mode
DEBUG = True

ALLOWED_HOSTS = [ "192.168.81.11" ]
CSRF_COOKIE_SECURE = False
SESSION_COOKIE_SECURE = False

OPENSTACK_HOST = "192.168.81.11"
OPENSTACK_KEYSTONE_URL = "https://%s/identity/v3" % OPENSTACK_HOST

WEBSSO_ENABLED = True
WEBSSO_INITIAL_CHOICE = "saml2"

WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("saml2", _("Security Assertion Markup Language"))
)
```

Relevant parts of /opt/stack/horizon/openstack_dashboard/local/local_settings.py
## References:
[^1]: [Introduction to Keystone Federation](https://docs.openstack.org/keystone/latest/admin/federation/introduction.html)
[^2]: [Using Keycloak with OpenStack](https://ivarclemens.nl/using-keycloak-with-openstack/)
[^3]: [DevStack](https://docs.openstack.org/devstack/latest/)
[^4]: [Stack overflow: Unable to install devstack in a Virtual Box](https://stackoverflow.com/a/72437577/24805424)
[^5]: [Stack overflow: Virtualbox enable nested vtx/amd-v greyed out](https://stackoverflow.com/a/57229749/24805424)

