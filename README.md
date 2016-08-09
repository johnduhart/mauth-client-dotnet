# Medidata.MAuth

Medidata.MAuth is a framework that provides support for authenticating web services and applications
with the Medidata HMAC protocol, MAuth.

## What is MAuth?

The MAuth protocol provides a fault-tolerant, service-to-service authentication scheme for Medidata and third-party
applications that use web services to communicate. The Authentication Service and integrity algorithm is based on
digital signatures encrypted and decrypted with a private/public key pair.

The Authentication Service has two responsibilities. It provides message integrity and provenance validation by
verifying a message sender's signature; its other task is to manage public keys. Each public key is associated with
an application and is used to authenticate message signatures. The private key corresponding to the public key in the
Authentication Service is stored by the application making a signed request; the request is encrypted with this
private key. The Authentication Service has no knowledge of the application's private key, only its public key.

## Medidata.MAuth Components

The framework is divided into separate semi-dependent NuGet packages to provide only the functionality which you need
in your application. Below you can find the description of each individual packages.

### Medidata.MAuth.Core

A core package for the MAuth protocol. This package contains the core functionality which used by the other MAuth
authentication protocol-specific components. This package also can be used standalone if you want to sign HTTP/HTTPS
requests with Medidata MAuth keys using the .NET _HttpClient_ message handler mechanism.

The package recommended in client applications where the application is required to issue MAuth-signed requests to an
MAuth-enabled service.

### Medidata.MAuth.Owin

This package contains an OWIN middleware to validate signed HTTP requests with the Medidata MAuth protocol.
The middleware communicates with an MAuth server in order to confirm the validity of the request authentication header.

Include this package in your OWIN-enabled application if you want to authenticate the incoming requests signed with
the MAuth protocol.

### Medidata.MAuth.WebApi

The package contains an HTTP message handler to validate signed HTTP requests with the Medidata MAuth protocol.
The handler communicates with an MAuth server in order to confirm the validity of the request authentication header.

Include this package in your WebAPI application if you want to authenticate the incoming requests signed with
the MAuth protocol.

## Get Started

Below you can find some information and examples on getting started using the framework.

### Installation

Depending on your needs you can install either the Core package only (for signing requests), or the Owin or WebApi
packages as well if you want to authenticate incoming requests.

The installation is as usual with NuGet.

For signing:

```
nuget install Medidata.MAuth.Core
```

Or in case of the authenticating, either

```
nuget install Medidata.MAuth.Owin
```

or

```
nuget install Medidata.MAuth.WebApi
```

For all of these you can use the _Visual Studio Package Manager_ as well.

The Owin and WebApi packages are dependent on the Core package, therefore it will be installed automatically in those
cases.

### Signing Outgoing Requests

In order to sign outgoing requests, an `MAuthSigningHandler` class is provided in the Core package. this handler
accepts an `MAuthOptions` instance which stores all the necessary settings for MAuth.

An example:

```C#
using Medidata.MAuth.Core;

public async Task<HttpResponseMessage> SignAndSendRequest(HttpRequestMessage request)
{
	var signingHandler = new MAuthSigningHandler(new MAuthOptions()
	{
		ApplicationUuid = new Guid("7c872d75-986b-4c61-bb17-f2569d42bfb0"),
		PrivateKey = File.ReadAllText("ClientPrivateKey.pem")
	});

	using (var client = new HttpClient(signingHandler))
	{
		return await client.SendAsync(request);
	}
}
```

The example above is creating a new instance of a `HttpClient` with the handler responsible for signing the
requests and sends the request to its designation. Finally it returns the response from the remote server.

The `MAuthOptions` has the following properties to determine the required settings:

| Name | Description |
| ---- | ----------- |
| **ApplicationUuid** | Determines the unique identifier of the client application used for the MAuth service authentication requests.  This uuid needs to be registered with the MAuth Server in order for the authenticating server application to be able to authenticate the signed request. |
| **PrivateKey** | Determines the RSA private key of the client for signing a request. This key must be in a PEM ASN.1 format. |

### Authenticating Incoming Requests with the OWIN Middleware

If your application implements the OWIN-specific pipeline, you can wire in the `MAuthMiddleware` provided by the
Owin NuGet package. The setting and usage is as follows (in the application's `Startup` class):

```C#
using Medidata.MAuth.Owin;

public class Startup
{
	public void Configuration(IAppBuilder app)
	{
		app.UseMAuthAuthentication(options =>
		{
			options.ApplicationUuid = new Guid("a419de8f-d759-4db9-b9a7-c2cd14174987");
			options.MAuthServiceUrl = new Uri("https://mauth.imedidata.com");
			options.AuthenticateRequestTimeoutSeconds = 10;
			options.HideExceptionsAndReturnForbidden = true;
			options.PrivateKey = File.ReadAllText("ServerPrivateKey.pem");
			options.Bypass = (request) => request.Uri.AbsolutePath.StartsWith("/allowed");
		);
	}
}
```

The middleware takes an `MAuthMiddlewareOptions` instance to set up the authentication:

| Name | Description |
| ---- | ----------- |
| **ApplicationUuid** | Determines the unique identifier of the server application used for the MAuth service authentication requests. This uuid needs to be registered with the MAuth Server in order to use it. |
| **MAuthServiceUrl** | Determines the endpoint of the MAuth authentication service. This endpoint is used by the authentication process to verify the validity of the signed request. |
| **PrivateKey** | Determines the RSA private key of the server application for the authentication requests. This key must be in a PEM ASN.1 format. |
| **AuthenticateRequestTimeoutSeconds** | An optional parameter that determines the timeout in seconds for the MAuth authentication request - the MAuth component will try to reach the MAuth server for this duration before it throws an exception. If not specified, the default value will be **10 seconds**. |
| **HideExceptionsAndReturnForbidden** | An optional parameter that determines if the middleware should swallow all exceptions and return an empty HTTP response with a status code Forbidden (403) in case of any errors (including authentication and validation errors). The default is **true**. |
| **Bypass** | Determines a function which evaluates if a given request should bypass the MAuth authentication. |

The **HideExceptionsAndReturnForbidden** parameter is useful (if set to **false**) when you have an exception handler
mechanism (for example a logger) in your OWIN pipeline. In this case the MAuth middleware won't swallow the exceptions
but will throw them with full stack trace and details of the problem - as every authentication errors will throw a
`Medidata.MAuth.Core.AuthenticationException` you can still return a Forbidden (403) HTTP status code in those
cases.
In the other hand, if you don't use any exception handling mechanism, it is recommended to leave this feature disabled
as setting this to **false** can possibly lead to exposing sensitive details about your application and the
authentication process. Leaving this parameter as **true** will result the middleware to return a Forbidden (403) HTTP
status code for every error without showing any details.

The **Bypass** function takes a `IOwinRequest` instance and should produce **true** as a result, if the given request
satisfies the conditions to bypass the authentication; otherwise it should result **false** therefore an authentication
attempt will occur. If no Bypass predicate provided in the options, every request will be authenticated by default.

### Authenticating Incoming Requests with the WebApi Message Handler

If your application does not use the OWIN middleware infrastructure, but it uses the ASP.NET WebAPI framework, the
WebApi package provides an `MAuthAuthenticatingHandler` which can be assigned to WebAPI routes or the global
handler collection in order to automatically authenticate incoming requests.

For a global registration (that is, use MAuth authentication for all requests), you can register the handler as below
(in your `WebApiConfig` class):

```C#
using Medidata.MAuth.WebApi;

public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		var options = new MAuthWebApiOptions()
		{
			ApplicationUuid = new Guid("a419de8f-d759-4db9-b9a7-c2cd14174987"),
			MAuthServiceUrl = new Uri("https://mauth.imedidata.com"),
			AuthenticateRequestTimeoutSeconds = 10,
			HideExceptionsAndReturnForbidden = true,
			PrivateKey = File.ReadAllText("ServerPrivateKey.pem")
		};

		config.MessageHandlers.Add(new MAuthAuthenticatingHandler(options));
	}
}
```

Alternatively, you can add the authenticating handler to a specific route when you define the route:

```C#
using Medidata.MAuth.WebApi;

public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		var options = // See the previous example

		config.Routes.MapHttpRoute(
			name: "Route1",
			routeTemplate: "api/{controller}/{id}",
			defaults: new { id = RouteParameter.Optional },
			constraints: null,
			handler: new MAuthAuthenticatingHandler(options)
		);
	}
}
```

In the examples above, the `MAuthWebApiOptions` instance has the same properties as the OWIN-specific
`MAuthMiddlewareOptions`.

## Frequently Asked Questions

##### What are the license terms for Medidata.MAuth?

The framework is licensed under the [MIT licensing terms](https://github.com/mdsol/mauth-client-dotnet/blob/master/LICENSE.md).

##### What is the current target .NET Framework version?

The current target is **.NET Framework 4.5.2** - this means that you have to use at least this target framework version
in your project in order to make Medidata.MAuth work for you.

##### Is there an ASP.NET Core support?

Currently we do not support the ASP.NET Core applications, but we built this framework with extensibility in mind,
so in the near future there is a possibility that we will add a new package with the ASP.NET Core support.

##### What Cryptographic provider is used for the encryption/decryption?

We are using the latest version (as of date 1.81) of the [BouncyCastle](https://github.com/bcgit/bc-csharp) library.
