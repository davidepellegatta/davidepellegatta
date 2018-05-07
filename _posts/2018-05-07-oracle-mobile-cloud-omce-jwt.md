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

On the "Policies" area is necessary to export your `policies.properties` file.