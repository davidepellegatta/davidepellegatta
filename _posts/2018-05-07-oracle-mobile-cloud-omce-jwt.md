---
title: Oracle Mobile Cloud & JWTs
layout: post
author: Davide Pellegatta
image: arctic-5.jpg
---

Oracle Mobile Cloud (OMCe) allows to integrate your mobile environments with a third-party identity providers (IdP). 

This post is about how to exchange your IdP JWT (and for instance one that  you've got throught an OpenId Connect implicit flow) for an OMCe one (and thus getting your users authenticated on OMCe).

I won't reproduce Oracle documentation that is already more then exhaustive. What I'd like to do is to clear up how to find what it's necessary, when it's needed. :)

## Overview

![OICD implicit flow with OMCe](/assets/img/2018-05-07-oracle-mobile-cloud-omce-jwt/oidc_flow_summary.png)

On the activity diagram above I've summarized the main steps necessary to authenticate a user with an OpenId Connect implicit flow on OMCe. 

The steps covered on this article are:

* how to pass an `id_token` to your mobile backend
* how to getting it succesfully validated


## JWT & IdP Setup

First thing is to get your JWT ready to be accepted by your mobile environment so some important considerations have to be done:

`sub` claim is - of course - used by OMCe to evaluate the identity of the authenticated user. It must be unique for that `token issuer`. It can be extremely useful to know that OMCe can be set up to evaluate a different claim (see references).

`aud` claim is a list of audiences to whom the issued JWT was intended for. In this example we will use an identity token (`id_token`) so the aud claim be set with the same client_id used for identifying your app with your IdP.

Normally OMCe expects to find the audience claim set with the url of your mobile back-end.  We will need to adjust this configuration.

A last consideration regards how your IdP signs its JWTs because according to it few settings on your Mobile Backend might change. 

Does your IdP implements the discovery standard standard? Does it have a JWKs endpoint? Are your SSL certificates signed by a valid CA? Sign down the answers to these questions: in the next chapter they'll be really useful.


## OMCe Setup

Well here it starts the trickie part. :)

Configurations for JWT exchange are valid for your whole instance. So if you have already some activities in progress you may want to consider to make your tests on a different environment.

OMCe stores JWTs exchange configurations on a policies property file. You can find on the Administration link on the right menu of your instance.

![OMCe Administration Panel](/assets/img/2018-05-07-oracle-mobile-cloud-omce-jwt/OMCe_administration_panel.png)

On the "Policies" area is necessary to export your `policies.properties` file. This file looks like this and it is better to read carefully all the alerts ;) :

```java
# ---------------------------------------------------------
# MCS Policies.
# ---------------------------------------------------------
# ! DO NOT EDIT THE HEADER !
# Snapshot from: oicdtest
# System version: 18.2.3-201804172244
# Data format: 2
# ---------------------------------------------------------
*.*.Admin_DcsConfig={}
*.*.Admin_FeatureConfiguration={ "DCS"\: true }
*.*.Admin_MigrationConfiguration={}
*.*.Analytics_ApiCallEventCollectionEnabled=true
*.*.Asset_AllowPurge=ALL
*.*.Asset_AllowTrash=ALL
*.*.Asset_AllowUntrash=ALL
*.*.Asset_DefaultInitialVersion=1.0
*.*.CCC_DefaultNodeConfiguration=8.9
*.*.CCC_LogBody=false
*.*.CCC_LogBodyMaxLength=512
*.*.CCC_MaxLoadPerCpu=1
*.*.CCC_MinFreeMemoryMegabytes=256
*.*.CCC_SendStackTraceWithError=false
*.*.Database_CreateTablesPolicy=explicitOnly
*.*.Database_MaxRows=1000
*.*.Database_QueryTimeout=20
*.*.Diagnostics_AverageRequestTimeErrorThreshold=6000
*.*.Diagnostics_AverageRequestTimeWarningThreshold=3000
*.*.Diagnostics_ExcludedHttpHeadersInLogs=Authorization,Cookie,oracle-mobile-uitooling-password,Oracle-Mobile-Social-Access-Token
*.*.Diagnostics_LongRequestCountErrorThreshold=10
*.*.Diagnostics_LongRequestCountWarningThreshold=0
*.*.Diagnostics_LongRequestThreshold=8000
*.*.Diagnostics_PendingRequestErrorThreshold=30
*.*.Diagnostics_PendingRequestWarningThreshold=15
*.*.Diagnostics_RequestCountErrorThreshold=10
*.*.Diagnostics_RequestCountWarningThreshold=0
*.*.Diagnostics_RequestPercentageErrorThreshold=10
*.*.Diagnostics_RequestPercentageWarningThreshold=1
*.*.Logging_Level=800
*.*.Network_HttpConnectTimeout=20000
*.*.Network_HttpPatch=METHOD
*.*.Network_HttpReadTimeout=20000
*.*.Network_HttpRequestTimeout=40000
*.*.Notifications_DeviceCountWarningThreshold=70.0
*.*.Routing_DefaultImplementation=system/MockService(1.0)
*.*.Security_AllowOrigin=disallow
*.*.Security_ExposeHeaders=
*.*.Security_IdentityProviders=[{"identityProviderName"\:"facebook","properties"\:{"graphApiUrl"\:"https\://graph.facebook.com/v2.5/"}}]
*.*.Security_IgnoreHostnameVerification=false
*.*.Security_TokenExchangeTimeoutSecs=21600
*.*.Sync_CollectionTimeToLive=86400
*.*.User_AllowDynamicUserSchema=false
*.*.User_DefaultUserRealm=Default(1.0)
# ---------------------------------------------------------
# WARNING: 
#
# The policies in this section are critical to MCS system operability.
# Editing these policies could severely impact MCS. Generally,
# you shouldn't need to modify them. If you must make changes,
# you should have a thorough understanding of each policy's function
# and the impact the changes will make to MCS. 
# ---------------------------------------------------------
*.platform/analytics(1.0).Routing_BindApiToImpl=platform/analytics(1.0)
*.platform/appconfig(1.0).Routing_BindApiToImpl=platform/appconfig(1.0)
*.platform/auth(1.0).Routing_BindApiToImpl=platform/auth(1.0)
*.platform/database(1.0).Routing_BindApiToImpl=platform/database(1.0)
*.platform/devices(1.0).Routing_BindApiToImpl=platform/devices(1.0)
*.platform/extended(1.0).Routing_BindApiToImpl=platform/usersExtended(1.0)
*.platform/location(1.0).Routing_BindApiToImpl=platform/location(1.0)
*.platform/sso(1.0).Routing_BindApiToImpl=platform/sso(1.0)
*.platform/storage(1.0).Routing_BindApiToImpl=platform/storage(1.0)
*.platform/sync(1.0).Routing_BindApiToImpl=platform/sync(1.0)
*.platform/ums(1.0).Routing_BindApiToImpl=platform/ums(1.0)
*.platform/users(1.0).Routing_BindApiToImpl=platform/users(1.0)
*.system/analyticsDataManagement(1.0).Routing_BindApiToImpl=system/analyticsDataManagement(1.0)
*.system/analyticsExport(1.0).Routing_BindApiToImpl=system/analyticsExport(1.0)
*.system/databaseManagement(1.0).Routing_BindApiToImpl=system/databaseManagement(1.0)
*.system/locationManagement(1.0).Routing_BindApiToImpl=system/locationManagement(1.0)
*.system/notifications(1.0).Routing_BindApiToImpl=system/notification(1.0)
```

In order to make everything work, we will need to add a custom property to it. 


## \*.\*.Security_AuthTokenConfiguration

The Security_AuthTokenConfiguration contains all the settings necessary to integrate your IdP with OMCe. 

The settings are passed to OMCe as a JSON value for `*.*.Security_AuthTokenConfiguration` property.  [Here](https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/authentication-omce.html#GUID-7716358B-BA49-4850-9D70-25A693F75DBA) a full reference to the JSON properties and values. In the following paragraphs will we try to set up the minimal configuration to make the process work.

```javascript
{
  "issuers": [
    {
      //map your IdP
      "issuerName": "https://your.issuer.com", 
			
      //says OMCe that your users are trusted and they don't need a local bind
      "virtualUserEnabled": true,
			
      //tells OMCe where to find your username
      "usernameAttribute": "sub",
			
      //mapped (In case of an id_token) your client_id
      "audience" : [
        "your_client_id"
      ],
			
      //tells OMCe where is your discovery document so that it can find your JWKs endpoint
      "jwks": {
        "discoveryUri": "https://your.issuer.com/.well-known/openid-configuration"
      },
			
      //instructs OMCe which default role will assigned to the authenticated users
      "defaultRoles" : [
        "customer_app_user"
      ]
    }
  ]
}
```

### Tweaks

#### Signing Certificate, SSL and JWKs

With the above configuration is assumed that:

* your IdP is exposed with a valid and it has been issued by a trusted CA;
* your IdP implements the discovery protocol ([http://openid.net/specs/openid-connect-discovery-1_0.html](http://openid.net/specs/openid-connect-discovery-1_0.html));
* your IdP has a `jwks` enpoint ([https://tools.ietf.org/html/rfc7517](https://tools.ietf.org/html/rfc7517)).

Exactly the kind of things that you don't have in a dev or a test environment (in particular SSL "cool" certificates)! 

To set up the environment in order to allow untrusted certificates it's neccessary to:

* add them (including all SSL chain) to OMCe
* and associate them to your issuer.



#### Adding SSL certificates to OMCe

To add a new certificate to your OMCe instance you have to open the administration tab (on the page show above) and select the "Keys and Certificates" button. Once the dialog appear select "Web Service and Token Certificates".


![OMCe Administration Console Keys and Certificates](/assets/img/2018-05-07-oracle-mobile-cloud-omce-jwt/OMCe_administration_keys_and_certificates.png)

Here you can upload your certificate and - eventually - _all the certificates in the SSL chain_ (CA Certificate, Intermediate CA Certificate, etc.). The newly uploaded certificates will be ready in some minutes.

Once the provisioning is completed you have to pair your certificate with your issuer. On the Keys and Certificates popup, select the "Token Issuers" tab and add a new issuer as it is specified on the `issuerName` of your `Security_AuthTokenConfiguration configuration`.

![OMCe Administration Token Issuer](/assets/img/2018-05-07-oracle-mobile-cloud-omce-jwt/OMCe_administration_token_issuer.png)

Once created, you have to associate the new configuration with your SSL certificate. 

The next step regards, again, your Security_AuthTokenConfiguration property:

```javascript
{
  "issuers": [
    {
      //map your IdP
      "issuerName": "https://your.issuer.com", 
			
      //says OMCe that your users are trusted and they don't need a local bind
      "virtualUserEnabled": true,
			
      //tells OMCe where to find your username
      "usernameAttribute": "sub",
			
      //mapped (In case of an id_token) your client_id
      "audience" : [
        "your_client_id"
      ],
			
      //pairs OMCe configuration with your IdP
      "certificateSubjectNames" : [
        "CN=your.issuer.com,OU=Your Department,O=Acme Corporation,L=Hill Valley,ST=California,C=US"
      ],
			
      //instructs OMCe which default role will assigned to the authenticated users
      "defaultRoles" : [
        "customer_app_user"
      ]
    }
  ]
}
```

You have to substitute the discovery configuration (`jwks` prop) with the `certificateSubjectNames` array that will contain the name of the certificates that you have just setup in the administration console.

## Try it out

At the page [https://docs.oracle.com/en/cloud/paas/mobile-suite/rest-api-platform/op-mobile-platform-auth-token-post.html](https://docs.oracle.com/en/cloud/paas/mobile-suite/rest-api-platform/op-mobile-platform-auth-token-post.html) are available some useful samples about how to perform the token exchange.

I've collected some errors that might be useful while you will debug yours :D

### Token Expired

```javascript
{
    "error": "invalid_grant",
    "error_description": "User assertion from issuer https://your.issuer.com has expired."
}
```

Remember that in your JWT Token it has to be present an `exp` claim! And it should be valid!

### SSL Certificate Problems

```javascript
{
    "error": "invalid_grant",
    "error_description": "No matching public key found for issuer https://your.issuer.com has expired."
}
```

Ok. This can have multiple causes. 

I've crossed this error on the test environments. Going through the instance logs I've found out the the problem was all about my discovery enpoint. The SSL certificate wasn't valid so OMCe could't perform the SSL Handshake in order to retrieve the discovery document and thus the jwks doc.

### Security_AuthTokenConfiguration configuration problems

```javascript
{
    "error": "invalid_grant",
    "error_description": "Issuer https://your.issuer.com has not been configured."
}
```

One of the mistakes I've done regards the policies scoping: you can apply a certain property to a certain mobile backend, api, etc. (See here: [https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/omce-policies.html#GUID-C9533720-55F2-4646-8278-F58A8386988F](https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/omce-policies.html#GUID-C9533720-55F2-4646-8278-F58A8386988F)). Security_AuthTokenConfiguration can not be scoped in this way and you need to specify it as a instance wide policy. It means that you have to declare it this way:`*.*.Security_AuthTokenConfiguration`. 

Despite this, you may want to make available your configuration just to a certain mobile backend. You can dothat specifying a `allowedMbes` property in your Json configuration.


## Docs References

#### JWT Tokens and Virtual Users

Everything is summed up here. But some resources are not really easy to find.

[https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/authentication-omce.html#GUID-BC160F1B-A50B-4B94-A655-5EE6EDC5BA7B](https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/authentication-omce.html#GUID-BC160F1B-A50B-4B94-A655-5EE6EDC5BA7B)

#### JWT Configuration Reference
[https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/authentication-omce.html#GUID-7716358B-BA49-4850-9D70-25A693F75DBA](https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/authentication-omce.html#GUID-7716358B-BA49-4850-9D70-25A693F75DBA)

#### OMCe Policy Names
[https://docs.oracle.com/en/cloud/paas/mobile-suite/manage/policies.html#GUID-5201BC18-3843-435D-B8D1-AB4D200EF2A0](https://docs.oracle.com/en/cloud/paas/mobile-suite/manage/policies.html#GUID-5201BC18-3843-435D-B8D1-AB4D200EF2A0)

#### OMCe Policies and Values
[https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/omce-policies.html#GUID-C9533720-55F2-4646-8278-F58A8386988F](https://docs.oracle.com/en/cloud/paas/mobile-suite/develop/omce-policies.html#GUID-C9533720-55F2-4646-8278-F58A8386988F)

#### Test It!
[https://docs.oracle.com/en/cloud/paas/mobile-suite/rest-api-platform/op-mobile-platform-auth-token-post.html](https://docs.oracle.com/en/cloud/paas/mobile-suite/rest-api-platform/op-mobile-platform-auth-token-post.html)