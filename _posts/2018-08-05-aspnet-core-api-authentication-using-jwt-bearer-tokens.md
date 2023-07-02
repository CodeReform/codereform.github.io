---
layout: post
title: ASP.NET Core API authentication using JWT bearer tokens
date: 2018-08-05 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ".NET Core"
- C#
tags:
- ".NET Core"
- C#
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2018/08/05/aspnet-core-api-authentication-using-jwt-bearer-tokens"
---
# ASP.NET Core API authentication using JWT bearer tokens

![](https://miro.medium.com/0*SaA65QL6u5FRUTc9)

What is OAuth 2.0 and how its flows can be applied for securing my applications? What does a token do and how it is useful in securing API's? Is there any way to implement all these nice and easy in ASP.NET Core? In this post I will cover these topics, by first discussing about why token based security is so successful in security scenarios, and the OAuth protocol play in this. We'll see more closely one of OAuth flows, the client credentials flow and implement it to secure an ASP.NET Web API application.

## Source code

Source code outlined in this article can be found on [GitHub](https://github.com/gdyrrahitis/bpost-aspnetcore-jwt).

## What is token based security

In token based security, a client application exchanges a token in order to access a protected resource. Usually we have four entities to describe this model. Following is an abstract protocol flow, with all the entities listed, taken from [RFC 6749](https://tools.ietf.org/html/rfc6749).

![](https://miro.medium.com/0*6HuDVsDZTdxp7h_X)

The resource owner (user), is the owner of the protected resource. Being the owner, means that he holds all the proper keys to access that resource, usually a username and password.

The client application is interacting directly with the resource owner and requires from that entity to authorize in order to access a protected resource. This can be any kind of application, authored even by a third party. But it doesn't know anything about the resource owner username or password, what it knows is that it needs something to exchange for some information about a specific request, the credentials remain unknown for this application, only the authorization server knows about them.

The authorization server is in charge of issuing tokens and ensuring that the token that was issued is still active. That server keeps track of the resource owner credentials and is, usually, the only one in the system that knows about them. It is totally isolated from the rest of the system.

The resource server, which is an application that serves the protected resources to the client application, is an entity that knows about the authorization server. It has established a trusted relationship with the authorization server and knows that the token issued is valid, or in other cases it even asks the authorization server to validate that token. The trusted relationship lies usually on the cryptography technique that is used to sign/validate a token, so that an evil entity cannot forge it.

This kind of security is very popular in web services, especially with RESTful API's, as not only this technique is highly secure, but it is easy to implement, due to the protocol nature, in various kind of applications, like desktop, mobile, web, whereas other kind of security methods, like cookies cannot apply, as they are only available in browser based-application scenarios.

## Tokens

The trusted authority, i.e. the authorization server, issues signed tokens by using a private key. These tokens contain information about the user, with various claims listed in them.
 We mostly refer to them as JSON Web Tokens, a special token format that is very popular in token based authentication. JWT's are essentially JSON data, encapsulated in a manner that makes it easy for consumers to read the data in a standard format.

The authorization server is the entity responsible of signing those tokens and it does that by using a private key for this purpose, which makes it very hard for an attacker to forge the token.
 The resource server is responsible of validating that token, by checking against the key shared by the authorization server. If the token has changed in between, then it rejects it as invalid.

Let's see how these make sense, by looking at the format of a JWT. I will use this very useful JWT debugger, [https://jwt.io](https://jwt.io/) to debug my token.

![](https://miro.medium.com/0*Uijp1iPBrtnn1xWm)

On the left hand side, you can see the raw format of the token. It is Base64 encoded (actually it is Base64URL encoded, which is kinda the same as Base64 but it is friendlier to URL's as it is not using reserved URL characters, look at a related post from Brock Allen [here](https://brockallen.com/2014/10/17/base64url-encoding/)) and you might notice it is broken down in three sections, separated by dot (.).

The first section is called the header and it includes information about the algorithm and the token type. The next section is called the payload and is the most interesting part of a JWT, as it contains all the data the client application needs and also claims about the authenticated user. The last section is called the signature, usually referred to as a MAC (Message Authentication Code). It can only be produced by someone in possession of both the header and payload and a given secret key. The MAC is produced by hashing (or encrypting, based on the algorithm that's chosen) the payload and the header, which is the responsibility of the authorization server. Usually, the authorization server chooses the cryptography algorithm to use in order to sign the tokens, which affects the token validation process, meaning it affects on how the resource server is configured as well. There are two types of JWT tokens, one is hashed SHA-256 and the other is encrypted via RS256.

The resource server can verify if the token is valid, by utilizing the verification signature against the secret/public key that it holds.

This is of course a very brief, touching the surface, overview of how a JWT is signed and verified across servers and what actions are performed behind the scenes, but I will not focus into that in this post, probably is a topic that deserves its own blog post.

## OAuth 2.0 and OIDC

Open authorization protocol, or OAuth, is a protocol that provides industry standards to build enterprise-ready secure applications, incorporating the entities mentioned before, resource owner, resource server, authorization server and client. Using this protocol, we can create appropriate flows to secure our applications regardless of the application type, being web, desktop or mobile. Access is common for all flows and is provided by exchanging tokens for protected resources, which OAuth specification refers to as access tokens, like a JWT, which gives to the client application limited access to the protected resources, in terms of token lifetime and scope.

Along with OAuth protocol goes the Open Id Connect protocol, which adds an extra layer on top of OAuth, with focus on verifying the identity and to obtain basic profile information for the authenticated end-user.

The OAuth protocol has various flows that can be useful for different types of applications, let's discover these flows and focus on the client credentials flow for this blog post.

## Flows

The authentication flows or grants, dictate the process on how a client application can receive an access token from the authorization server. Each flow is appropriate for different scenarios, not all flows are appropriate for all kinds of applications. We have two types of redirect flows and two types of credential exchange type of flows.

### Redirect flows

- **Implicit grant flow**. This is one of the most popular approaches, the resource owner is redirected to the authorization server and logins there. After a successful login, user is redirected back to the client application, where they get their access token.

- **Authorization code flow**. This is similar approach to the above, with one twist. Instead of getting an access token when redirected back to the website, we simply get an authorization code, which can be used to trade for an access token. This mechanism adds an extra layer of security, mitigating MIM attacks.

Learn more about implicit vs authorization code flow in this [great answer on SO](https://stackoverflow.com/questions/13387698/why-is-there-an-authorization-code-flow-in-oauth2-when-implicit-flow-works-s).

### Credential flows

- **Resource owner password credentials flow**. In this case, the client application trades the username and password for access to the API. Password and username are included in the request. It is recommended to use this kind of flow over HTTPS, as in HTTP the credentials are transferred in plain text, so someone eavesdropping the network could easily steal them. This flow is more recommended in internal applications, mitigating the risk for attacks/exposure of credentials.

- **Client credentials flow**. This is the flow we are going to focus in this blog post. In this flow, we trade our client Id and secret for an access token. More for this flow in the next section.

## Client credentials flow

As mentioned earlier, in this case we trade the combination of client Id and secret for an access token.

The authorization server has created a client Id for each of his known clients and has a secret key associated with each of them. That secret key might be different for each client, but the combination of the client Id and the secret is enough to identify a registered client application. This flow is particularly useful when it is desired to access resources on an API without a particular user connected, mostly this approach is better in server-to-server scenarios, when interconnected internal applications within a system need to authenticate without any UI interaction.

Note that the client credentials grant type must only be used by confidential clients. Let's see the flow in the following diagram.

![](https://miro.medium.com/0*qkEwW5FYKJXJFesY)

1. The client authenticates with the authorization server and requests an access token by providing a client Id and secret.

2. The authorization server authenticates the client and if the request is valid, issues an access token.

Now that we know what is client credentials flow, let's move to an application framework that abstracts all trivial steps away and makes it easy to create an authorization server that supports this kind of flow.

## IdentityServer4

It is a security framework for ASP.NET applications, providing out-of-the-box features on OIDC and OAuth. Using this framework, you can easily create a custom fully-fledged authorization server, with appropriate implementation of the OAuth and OIDC protocols. As you will see later in this post, creating an authorization server that has client credentials flow configured is nothing more than few lines of code. If you tried to do this manually, it would be very cumbersome, as implementing these protocols properly is not an easy task.

IdentityServer4 has specific [terminology](http://docs.identityserver.io/en/release/intro/terminology.html) for the entities that are involved in a secure transaction. We'll look only at the entities that make sense for this tutorial for now, you can find out more on the official documentation about [terminology](http://docs.identityserver.io/en/release/intro/terminology.html).

- **Resources**. **Resources** are something you want to protect with IdentityServer, such as APIs. For the latter, IdentityServer4 models them using the ApiResource entity.

- **Client**. A client is a piece of software that requests tokens from IdentityServer. A client must be first registered with IdentitySever before it can request for tokens. IdentityServer4 models a client with the **Client** entity.

It comes as a Nuget package which you can install in your web application. Please find the official documentation website [here](http://docs.identityserver.io/en/release/).

## Sample application

I have created an ASP.NET Core Web API with a standard ValuesController, which just returns an array of strings via its GET endpoint. I would like this endpoint to be secured, thus I have created another web application, which is going to be my authorization server and I will use the client credentials flow for this example.

I will start with the authorization server.

### Authorization server

I've created a new ASP.NET Core application and installed the IdentityServer4 Nuget package. Now it's time to create all the required entities. I need an ApiResource to describe the resource server (Web API application) in my system and also I need to register a new client, which can access my RESTful API. For this reason, I have created the following class.

{% gist 2d7f8c3343008f26d3253df9bf807ad8 %}

I have created an instance of ApiResource, with the name "auth.web.api". This describes my protected Web API and the class comes from the IdentityServer4 package.
 Next, I have created a new instance of a Client, a class that IdentityServer4 provides to describe an entity that can request access tokens. I have to provide a unique client Id for the client, in this case I want to create a client that can access APIs, so I call it "ApiClient".

For grant types, we want to authenticate using the client credentials flow, so I am using the same with GrantTypes.ClientCredentials. Next, I define a new client secret for this client and finally, the allowed scopes. This is important and it defines which API scopes this client can access. I have already defined an API resource earlier, so I am using the same name, "auth.web.api".

Last piece for the authorization server is to setup IdentityServer in ASP.NET. In order to set it up, the framework provides an extension method for the IServiceCollection, the AddIdentityServer which adds it to the ASP.NET Core DI. Now I can configure it as I please, with the following being the current configuration.

{% gist f2792d235aa2a6ab3dad7fd1f0897562 %}

At the beginning of this post, I briefly talked about JWT tokens and signing algorithms that they use. The method AddDeveloperSigningCredential is a helper method for local development and should not be used in production. What it does, is to setup RS256, generating a temporary RSA key. It is important to setup your signing algorithm, else IdentityServer will not be able to sign any token.

Then, I add the in-memory API resources and clients, using the appropriate extension methods.
 Finally, I have to call the UseIdentityServer extension method in Configure, in order to add it within the ASP.NET Core pipeline. Lastly, I have to take note on the application URL, as it will be useful when I'll setup authentication for my RESTful API.

The authorization server is ready, I can test it later, let's first visit the Web API and configure it properly.

### Web API

First, I will visit the Startup.cs and setup authentication there, securing the API with JWT tokens and OIDC. The call to AddAuthentication adds the required authentication services in the pipeline.

{% gist ccec437035db44b09471cbeb469217a3 %}

My authorization server signs JWT tokens, so I need to setup my authentication mechanism to use JWT bearer tokens, thus the call to the AddJwtBearer method. Of course, in order for this to work, I need to provide some basic configuration.

- **Audience**. This describes the access scope, the resource server that should accept the token. In authentication server we named that API resource as "auth.web.api".

- **Authority**. The URI of the identity provider that issues the token.

Also, notice that I am configuring the default authorization policy, telling the system that unless any other authorization policy is explicitly required, this should be the default. I need to setup the policy scheme to JwtBearerDefaults else the application will not have any default authentication scheme and it will fail at runtime.

Finally, I use the Authorize attribute to protect the API resource.

{% gist d843a40189a3f0b0717f2530edc48bb7 %}

### Testing with Postman

Let's see first, if the RESTful API is properly secured. It is up and running at https://localhost:44324, so making a request at /api/values would return a 401 Unauthorized response.

![](https://miro.medium.com/0*rVHqfXO3WjZazWh8)

It's time to test the authorization server, at https://localhost:44364, and if the Web API is indeed secure with the client credentials grant. But first, I have to receive an access token. How can I do that, what's the endpoint that I need to reach to? Fear not, cause IdentityServer provides an OIDC discovery endpoint, which can be used to retrieve metadata about the authorization server, via the /.well-known/openid-configuration path, relative to base address. As you can see below I can get more than the token endpoint, I can get info on the claims that are supported, the grant types and many more.

![](https://miro.medium.com/0*RUxVUVIFqf2Ph-Dh)

Now that I know where I can reach to request for a token, I can proceed with the actual request. This is a POST request to the /connect/token path, relative to base address, with data in the body as x-www-form-urlencoded. I am using client credentials flow, so it is required to provide three key-value pairs in this form.

1. **client_secret**. This is the client secret defined in the authorization server.

2. **client_id**. This is the unique client Id defined in the authorization server.

3. **grant_type**. Value is _client_credentials_.

![](https://miro.medium.com/0*Uw2IP00J2gMCwkoO)

Making this POST request, I've received an access token, which I can use to authenticate. I will now make a request to the protected resource, but I also need to include the Authorization header and provide the bearer token I received earlier.

![](https://miro.medium.com/0*g4HuYlBNRft-l4de)

I can now receive the expected response and not a 401.

## Summary

In this post we discovered the token based authentication using tokens in ASP.NET Core with [OAuth](https://oauth.net/2/) and [OIDC](http://openid.net/connect/). We've used the [IdentityServer4](http://docs.identityserver.io/en/release/) package to create a custom authorization server and grant client credentials access to a RESTful API. This whole operation was just a few lines of code, which demonstrates [IdentityServer4](http://docs.identityserver.io/en/release/) and ASP.NET Core power to secure applications via an easy and sophisticated API. Finally, we've tested the authorization server and access to the API via [Postman](https://www.getpostman.com/).

---

If you liked this blog, please like and share! For more, follow me on Twitter [@giorgosdyrra](https://twitter.com/giorgosdyrra).
