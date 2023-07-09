---
layout: post
image: https://miro.medium.com/1*P7x-_0XfQz6CVmMY_QAv0w.png
title: Build a secure Angular client using ASP.NET Core and OAuth
date: 2018-08-19 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- "Angular"
tags:
- "Angular"
- ".NET Core"
- "OAuth"
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2018/08/19/build-a-secure-angular-client-using-aspnet-core-and-oauth"
---
# Build a secure Angular client using ASP.NET Core and OAuth

![](https://miro.medium.com/1*P7x-_0XfQz6CVmMY_QAv0w.png)

What is the resource owner password credentials grant? How can I secure my Angular client using OAuth and JWT bearer tokens? In this post I will focus on the resource owner password credentials grant, a different kind of credential flow supported by the OAuth protocol, and how it can be used to secure certain resources on an Angular application. Similarly to [previous post](https://medium.com/@giorgos.dyrrahitis/asp-net-core-api-authentication-with-jwt-140a4858b5bd), I will create the authorization server from scratch, then the resource server, a simple ASP.NET Core RESTful API, and finally the Angular 6 application, with all the bits and pieces required to prevent unauthorized access.

## Source code

Code outlined in this article can be found on my GitHub [repository](https://github.com/gdyrrahitis/bpost-aspnetcore-angular).

## OAuth flows (again)

In [previous](https://medium.com/@giorgos.dyrrahitis/asp-net-core-api-authentication-with-jwt-140a4858b5bd) post, I briefly presented the available OAuth flows, with more focus on the client credentials authorization grant. To refresh our memory, an authorization grant or OAuth flow, is a credential representing the resource owner's authorization via an access token, with OAuth supporting 4 different flows, but also allows the extension of the current specification with custom flows.

In this post I will focus on the resource owner password credentials flow or else password grant. My goal is to go through the specification, understand the common usage of this grant type and finally implement a system the would allow a user to login with his/her credentials and access a protected resource in an SPA application with Angular.

## Resource owner password credentials flow

There are two kinds of credential flows, one is client credentials and the other is resource owner password credentials (or ROPC). Using the ROPC flow, the credentials (i.e. username and password) of a resource owner (i.e. user) can be exchanged for an access token in one request. This grant type should only be used when there is a high degree of trust between the resource owner and the client and other authorization grants can't be applied. OAuth protocol has become very popular because its goal is to solve the security risk of sharing user credentials with client applications, especially third-party, but this flow kinda seems to contradict the protocol's claims.

According to [RFC-6749](https://tools.ietf.org/html/rfc6749#section-4.3), third-party applications should never be allowed to use this grant, as it defeats OAuth protocol purposes, leaking critical information to untrusted client applications. So when should be used? Commonly, this grant type is used in scenarios where the client application is part of the current system, thus it is allowed to know some information about the resource owner.

Another use case can be the migration of existing clients that use direct authentication schemes, such as HTTP Basic or Digest authentication to OAuth, as the steps required for authentication are very similar in these scenarios. That's valid, as this grant type actually resembles the direct authentication schemes mentioned above, so migrating to OAuth shouldn't be a big deal for these kind of systems.

Finally, consider using this grant type when nothing else is available and if that's the case, always use HTTP over transport-layer security (TLS), so risks like man-in-the-middle attacks can be mitigated. Remember, when this grant type is used, username and password are included in the request, so in a non-secure HTTP scenario, an attacker can easily steal user's credentials.

The following figure depicts the password grant and the OAuth dance between entities involved in this flow.

![Resource owner password credentials flow](https://miro.medium.com/0*4qalG2BZ0xbRh7cc)

### The authorization request

The request for token should be in a specific format when using the ROPC flow and should contain the following parameters in the request body. Note that the request is a POST and the body should be in x-www-form-urlencoded format. All are required, except of the scope parameter.

- **username**. The resource owner **username**.

- **password**. The resource owner **password**.

- **grant_type**. Discriminates the grant type used. For ROPC is **_"password"_**.

- **scope (optional)**. Defines the scope of the authorization request.

- **client_id**. The issued client Id.

- **client_secret**. The issued client secret.

The last two (client_id & client_secret) should only be used when the client is secured and has issued a secret by the authorization server. When that's the case, these two are required as well.

## Let's start building

Now that we know what is the password grant and its OAuth dance, it's time to start building a system that would require from a user to login with his credentials in order to view a resource in a single page web application. I will start with the authorization server, the one that will sign and issue the access token and allow access to a user based on valid credentials.

Then, I will move to the resource server, which is going to be a simple ASP.NET Core Web API, with one controller named PetsController and a single GET action, which returns a collection of pet names.

Finally, I am going to setup an Angular 6 client using Angular CLI, setup the required components, services and routes, secure specific routes using route guards, create a login page where the user can enter his credentials and HTTP interceptors to include the access token for each authenticated request. As an end result, the user will be able to navigate to the pets route in Angular client and view a list of pet names, fetched from the RESTful API.

## Authorization server

For the authorization server I will use, once again, the IdentityServer4 NuGet package, which simplifies greatly such scenarios of custom identity providers.

I'm using dotnet CLI to install IdentityServer4 package to the authorization server application, which is an empty ASP.NET Core web application.

{% gist 1f31743fb0c4c4679491d99b5bee18b6 %}

The setup is pretty straightforward and very similar to the one presented in previous post. To summarize, I will need to setup the signing credentials, so for this simple example I will use the developer signing credentials that IdentityServer4 provides, I will also need an API resource, a client to correlate with that API and a user with username and password, which will be used while in ROPC flow. Following is that setup, translated from requirements to code.

{% gist 2cf89e8d40f72ec522379fdc60282815 %}

I have added a test user, using the TestUser class, included in the IdentityServer4 framework, with some very trivial username and password credentials. The SubjectId is the name identifier of the user and I've also included a name and email claims for the same user.

The API resource is also trivial and follows the same pattern as last time. As a second parameter, I've passed all the claim types that should be included in the JWT token. In this case, this API allows the Name and Email claims to be listed in the JWT token. These two are defined in TestUser‘s claims.

The client seems a bit different since last time. One difference is the AllowedGrantTypes, which now uses the ResourceOwnerPassword grant type. Something new also is the AllowedCorsOrigins property, which sets up the allowed origins for JavaScript clients, with the origin listed here being the URL of the Angular client, which I am going to demonstrate shortly.

Finally, in the Configure method, I'll need to include IdentityServer in the ASP.NET Core pipeline.

{% gist 665eaf2830e20324fe8d400406e389e2 %}

With authorization server in place, it's time to configure the resource server and expose the /pets endpoint.

## Resource server

I'll quickly setup a controller that exposes a GET endpoint, which returns a collection of pet names. I'll secure it, using the AuthorizeAttribute to decorate the controller.

{% gist 84a7139ebd66ece3da58a9776787e742 %}

Finally, I'' need to setup the authentication middleware to use JWT bearer tokens, in exactly the same way I did in the previous post for the client credentials grant.

{% gist dbc85c4689c241e791bbe524e117ea30 %}

If you have been following along, the setup is similar from last time, I'm adding the authentication middleware and calling the AddJwtBearer extension method to register my authorization server and define the API scope, that's what's needed to validate the access token.

Finally, in the Configure method, I'll need to register the authentication configuration and also enable CORS, as this API will be accessed via a JavaScript client.

{% gist 5c3dcc7d486a94cd412f0bd1c732e223 %}

With this last piece, backend is ready, the only thing left is to setup the Angular application.

## Angular client

### Let's try imagine what is needed

It's time to build the front end client application, however, before doing so, it's probably wiser to list all the requirements for this application and possible use cases.

So, we want an application where the user will navigate to a specific route and view a list of pet names. So this is a component, I will name it pets component. However, this page should be protected, so I will need to create a route guard, which responsibility is to verify if the user is authenticated or not. If user is not authenticated, then it navigates to a login page. For the login page, we'll need a new component, that will render a reactive form in the template and try to perform authentication, upon submit. We'll need an Angular service to retrieve information about user's authentication status and for performing the authentication dance with the authorization server. Finally, we'll need an Angular HTTP interceptor, which is going to add the Authorization HTTP header, with the bearer token to each request.

To keep the post short, I will demonstrate only the most interesting parts of the application, for full code, please refer to the GitHub [repository](https://github.com/gdyrrahitis/bpost-aspnetcore-angular).

To scaffold and build this application I'm using [Angular CLI](https://cli.angular.io/), a command line interface for Angular.

### Protected page

I will start with the protected page and create a component named pets. The Angular CLI command for this is

{% gist e5169a516428f3199af839d534222978 %}

Code inside the component is really simple, I just want to make a GET request to the RESTful API and retrieve a collection of pet names. Template is even simpler, so I won't bother show its details.

{% gist 50b444e218736c0156717ba614748792 %}

Component is ready, but I need to find a way to prevent access to unauthorized users. I will need a route guard, but before I create the guard, I will create a service first, which is going to verify if a user is authenticated and also perform the authentication itself.

### Authentication service

I've created a new service with the following Angular CLI command.

{% gist 29f1d740be2b9f73a66252669bc0cd1f %}

Next, I will add the required code to perform the actions described earlier.

{% gist cb465fb5230044ddf8dfd04b7b0a1021 %}

I've omitted most of the code inside, I would like to go through it piece by piece. In code above, I've injected the Angular's HttpClient and the JwtHelperService, a service coming from the @auth0/angular-jwt package. It provides nice helpers and that service can help me identify if the token has expired. I will start with the authenticate method, which is responsible in authenticating the resource owner using the ROPC flow, and then proceed to rest of the methods.

{% gist 6705b56cf2e6671fe6fdedece4fe762b %}

This method receives the username and password of the user in its signature. First, I create the HTTP headers collection, including only the Content-Type header for this request. Next, I am creating the body, which conforms with the OAuth 2 standards for the password grant. I am using the URLSearchParams class to construct the body just like a query string. Notice, when calling the post method, I convert it to a string. Also notice that I am using the rxjs pipe method and only one pipable operator, which is map, to persist the JWT token in the localStorage as soon as I retrieve it. This might not be the best practice, to store the token in local storage, because the application might be exposed to XSS attacks, but I'll use this technique for now.

Next, I want a method which can verify if the user is authenticated or not. That's simple, we'd only need to retrieve the token from localStorage and verify it's not expired.

{% gist 68731aed37452e45118fe6b62ae44bb4 %}

Code in the isAuthenticated method is a direct translation of the requirements into code. The JwtHelperService comes in handy here, as it can verify if the token has expired or not. Rest methods, getToken and logout, are pretty much self-explanatory.

A little side-note here, do not forget to register the JwtModule in your app module and configure it. It requires to setup the tokenGetter property, which value is a function that returns the stored token.

{% gist 25e34d8424f658f528410de18716e709 %}

### Angular route guards

Moving forward, I can now implement a route guard, using the Angular CLI command once again.

{% gist 2a999dd854780a23415a51598f2d40b9 %}

A route guard can be used to validate some router URL's and it can be either a service or a function. I will use a service type, which needs to implement the CanActivate interface. That interface has a method, canActivate, which resolves the current router parameters passed and returns a boolean. A false return value means the route can't be activated.

Now I need to work on the canActivate method of the guard, redirecting the user to the login page when not authenticated. To receive that piece of information, I will use the AuthService I created earlier.

{% gist 1810267b4f3cc5e56df1392d25a4f3a3 %}

Notice in the query parameters, in the navigate method, I'm adding the current router state URL in order to redirect the user back from the page where the request originated.

The route guard can be registered as a provider in the app module and to use it for a particular route, use the canActivate property of the route, like in the following example in app-routing.module.ts.

{% gist db6c1a6b067ec67ce0d4cca05ef3ab5a %}

### Login page

Next piece of functionality required is the login page, the route guard is redirecting to the /login route, which is handled by a component named AuthComponent which can be created with the following command.

{% gist 3e3900d6a5c91adc0991a88bb5169d4c %}

The implementation should be straightforward. I'll have to create a form for the user to enter a username and a password and upon submit I will authenticate the user, using the authenticate method of AuthService.

{% gist dce1aa4e61c43c54018d9756f85d24d1 %}

Notice that in ngOnInit two things are happening. First, I need to retrieve the returnUrl query string from the URL in order to use it later, when the user is successfully authenticated, to redirect back to it. Finally, I am creating a new reactive form with two form controls, username and password. Code in the template is trivial and I won't get into details, probably will save reactive forms specifics for another post. I will jump into the ngSubmit method though, which is called when the form is submitted.

{% gist 4f64d672aae313b65de75fbf9339b26c %}

First, I retrieve the values in username and password fields, which I then provide to the authenticate method of the AuthService that's injected. This returns an Observable, so I need to subscribe to it. If it resolves successfully, then user is redirected back to the returnUrl or home page. Otherwise, an error message is shown to the user.

### Angular HTTP interceptors

Finally, an HTTP interceptor is needed, in order to add the access token in each request header. Up to that point, the application can receive and store the token in the browser, but nothing interesting can happen until it sends that token to the resource server, otherwise the protected resource can't be fetched.

An HTTP interceptor is like a middleware, which is invoked on an HTTP request, trying to handle it, most likely transform it and pass it to the next handler in chain. In this case, we want to intercept the HTTP request and include the Authorization HTTP header with the bearer token as a value, only if the user is authenticated.

An HTTP interceptor can be implemented as a service and needs to implement the HttpInterceptor interface. The intercept method accepts an HttpRequest and an HttpHandler, which might be the next handler in chain to handle the HttpRequest. Create an interceptor as a class and manually add the interface and Injectable decorator. The CLI command would be

{% gist 2723657132174316e0a650499349fd6c %}

While the implementation is the following

{% gist 9441bf83b0ea193246f69462ea5b6b2d %}

If the user is authenticated, I retrieve the access token, then I clone the HttpRequest, updating it with the HTTP headers object passed in the clone method and finally I pass it to the handle method of the next handler in chain, if any.

What is left is to register the interceptor in the Angular DI as an HTTP interceptor, using the HTTP_INTERCEPTORS provider token.

{% gist 67d0e0fc860eef367b468d25f25aaef0 %}

That's it, all that's needed is implemented and the application is finally ready.

## End result

This is the JWT token the Angular application receives from the authorization server. Notice the payload, it includes all the required claims and also the name and email claims I've setup for TestUser earlier.

![](https://miro.medium.com/0*waPtQGXcnx_SgLzQ)

This is what the application looks like

<iframe src="https://cdn.embedly.com/widgets/media.html?url=http%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DCrI84sbjS8g&src=https%3A%2F%2Fwww.youtube.com%2Fembed%2FCrI84sbjS8g&type=text%2Fhtml&key=a19fcc184b9711e1b4764040d3dc5c07&schema=youtube" title="" height="480" width="854"></iframe>

## Summary

In this post we explored in detail the resource owner password credentials flow, and implemented an OAuth 2 client that uses it. Just like in previous post, setting up an authorization server was a breeze, all thanks to IdentityServer4 security framework for ASP.NET. Then, following similar steps we've setup the resource server as an ASP.NET Core Web API, securing certain endpoints with JWT bearer tokens. The last part with the Angular 6 client, involved quite a bit of work, introducing features like routing guards, HTTP interceptors, reactive forms, services and the Angular HTTP Client. All these together, made the secret sauce to finally secure the pets component, allowing us to make remote calls to the underlying API to fetch required data.

As mentioned earlier, ROPC flow might not be ideal in scenarios where you API is public, under no HTTPS encryption, as such scenario imposes great security risks. Also, this kind of flow might not be a great user experience for your users, as there is no concept of refresh token. When the token expires, user is automatically logged out, forced to login again in order to have access, which in some cases can be very annoying. Imagine you were filling the fields of a lengthy form and as soon as you clicked submit, the application logged you out, losing everything up to this point. Not cool.

The key behind designing your application is to thoroughly go through the application needs and evaluate against security strategies provided by protocols designed for this purpose, like OAuth. Once you have everything, the implementation would just be a direct translation of the requirements to code. The hard and critical part is to make the proper decision when it's about security, as this subject is very sensitive, especially for public APIs, where risks are higher.

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).

This post is part of the _ASP.NET Core Authentication series_.
