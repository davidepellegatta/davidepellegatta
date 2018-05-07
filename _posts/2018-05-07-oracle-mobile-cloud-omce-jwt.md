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

![OICD implicit flow with OMCe](/assets/img/oidc_flow_summary.png)

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

![OMCe Administration Panel](/assets/img/OMCe_administration_panel.png)

On the "Policies" area is necessary to export your `policies.properties` file. This file looks like this:

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

In order to make everything work we will need to add a custom property to it. 

## \*.\*.Security_AuthTokenConfiguration