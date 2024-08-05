---
title: Keystone Federation
categories: [OpenStack, Keystone]
date: 2024-08-05 23:04
tags: [notes]
media_subpath: '/posts/20240805'
---
Identity federation is the ability to share identity information across multiple identity management systems. In keystone, this is implemented as an authentication method that allows users to authenticate directly with another identity source and then provides keystone with a set of user attributes. This is useful if your organization already has a primary identity source since it means users don’t need a separate set of credentials for the cloud. It is also useful for connecting multiple clouds together, as we can use a keystone in another cloud as an identity source. Using [LDAP as an identity backend](https://docs.openstack.org/keystone/latest/admin/configuration.html#integrate-with-ldap) is another way for keystone to obtain identity information from an external source, but it requires keystone to handle passwords directly rather than offloading authentication to the external source.

Keystone supports two configuration models for federated identity. The most common configuration is with [keystone as a Service Provider (SP)](https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#keystone-as-sp), using an external Identity Provider, such as a Keycloak or Google, as the identity source and authentication method. The second type of configuration is “[Keystone to Keystone](https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#keystone-as-idp)”, where two keystones are linked with one acting as the identity source.

## **Service Provider (SP)**
A service provider is the service, providing the resource an end-user is requesting. For example, keystone provides tokens that we use on other OpenStack services.

## **Identity Provider (IP)**
An Identity Provider is the service that accepts credentials, validates them, and generates a yay/nay response. It returns this response along with some other attributes about the user, such as their username, their display name, and whatever other details it stores and you’ve configured your Service Provider to accept.

![normal keystone flow](normal_keystone_flow.png)

In a Normal Keystone, when a user comes for accessing the OpenStack services, he/she needs to go through the horizon dashboard for identity management, after which the authentication and authorization are both taken care of, by keystone.

We want to integrate keystone with keycloak for federated identity, so we are going to focus on the [keystone as a Service Provider (SP)](https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#keystone-as-sp) configuration.

---

Our endgoal is this: 
![saml2 websso keystone](saml2_websso_keystone.png)
This diagram shows a standard [WebSSO](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0-cd-02.html#5.1.Web%20Browser%20SSO%20Profile|outline) authentication flow, not one involving keystone. WebSSO is one of a few [SAML2.0 profiles](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0-cd-02.html#5.Major%20Profiles%20and%20Federation%20Use%20Cases|outline). It is based on the idea that a web browser will be acting as an intermediary and so the flow involves concepts that a browser can understand and act on, like HTTP redirects and HTML forms.

First, the user uses their web browser to request some secure resource from the Service Provider. The Service Provider detects that the user isn’t authenticated yet, so it generates a SAML Request which it base64 encodes, and then issues an HTTP redirect to the Identity Provider.

The browser follows the redirect and presents the SAML Request to the Identity Provider. The user is prompted to authenticate, probably by filling out a username and password in a login page. The Identity Provider responds with an HTTP success and generates a SAML Response with an HTML form.

The browser automatically POSTs the form back to the Service Provider, which validates the SAML Response. The Service Provider finally issues another redirect back to the original resource the user had requested.

Federated horizon page
![federated horizon page](federated_horizon_page.png)

Keystone is the openstack identity service, it's responsible for identification on our openstack cloud. The way keystone does authorization is by role assignments, that is how the keystone models in the backend and does the exactly the way the keystone tries to do, to link a user disk that is federated coming to the cloud into a project and based on the openstack side. It also works a service provider in the federation workflow, meaning that keystone provides the results that users are trying to access

Our requirement for the federated workflow/feature is, all the endpoints must be created that means we need to have SSL enabled in our environment
![overview of keystone](overview_of_keystone.png)

KeyCloak is the upstream of RedHat Single Sign On (SSO) and in federation language, it acts as the  identity provider (IdP). It validates user credentials and sends back to keystone, that user credentials are in fact valid. We can say it is the middleware between OpenStack and the LDAP directory, it acts as a identity aggregator. It provides the user information to the openstack through keystone.
![overview of keycloak](overview_of_keycloak.png)

\[Horizon Dashboard, Keystone\] and \[Keycloak, FreeIPA(LDAP in our case)\] are two main components for the federation, mellon is the SAML speaker.

Keystone itself can't talk/speak SAML and it needs someone to translate SAML into something that keystone can read and translate into openstack attributes that openstack understands.

![federating through keycloak](federating_through_keycloak.png)

Attribute Pipeline: LDAP->IdP->Mellon(SAML speaker)->SP
![attribute pipeline](attribute_pipeline.png)

References:
- [Using Keycloak with OpenStack](https://ivarclemens.nl/using-keycloak-with-openstack/)
- YouTube Video: [Federation with keycloak and FreeIPA](https://youtu.be/1tWiJZZfrPo?si=xfIdAkwaZzWSeMoD)
- Manual: [Federation with keycloak and FreeIPA](https://access.redhat.com/solutions/3010401)
- [Introduction to Keystone Federation](https://docs.openstack.org/keystone/latest/admin/federation/introduction.html#what-is-keystone-federation)
- [Configuring Keystone for Federation](https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#keystone-as-a-service-provider-sp)
- [Password, Session, Cookie, Token, JWT, SSO, OAuth - Authentication Explained - Part 1](https://blog.bytebytego.com/p/password-session-cookie-token-jwt)

