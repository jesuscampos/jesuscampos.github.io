---
layout: post
title: Autenticaci�n en ASP.NET Core
author: Jes�s
tags:
- ASP.NET
- Schemes
- Esquemas
- ASP.NET Core
- Startup
- C#
- Autenticaci�n
- Authentication handler
- Middleware
- Net Core
---
- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- Parte VI. Nota sobre redirect uri's

## Nota sobre *redirect uri's*
En una autenticaci�n remota existen dos *redirect uris* que no debemos confundir:

##### 1. La redirect uri a la que la **aplicaci�n cliente** debe redireccionar tras un login exitoso
Suele ser la url a la que el usuario intent� entrar y la raz�n por la que el **Challenge** se desencaden�.

El middleware establece en el pipeline informaci�n sobre la llamada original en una de sus [Features](https://docs.microsoft.com/es-es/aspnet/core/fundamentals/request-features?view=aspnetcore-3.1) para que est� disponible en el handler.
```csharp
// AuthenticationMiddleware
context.Features.Set<IAuthenticationFeature>(new AuthenticationFeature
{
    OriginalPath = context.Request.Path,
    OriginalPathBase = context.Request.PathBase
});
```
M�s adelante esta informaci�n es utilizada por el handler para crear la **propiedad de autenticaci�n** `RedirectUri`:
```csharp
// AuthenticationHandler (base de todos los handlers)
protected PathString OriginalPath => Context.Features.Get<IAuthenticationFeature>()?.OriginalPath ?? Request.Path;
protected PathString OriginalPathBase => Context.Features.Get<IAuthenticationFeature>()?.OriginalPathBase ?? Request.PathBase;

// Por ejemplo OAuthHandler, aunque otros handlers tanto remotos como no remotos se comportar�n igual
if (string.IsNullOrEmpty(properties.RedirectUri))
{
    properties.RedirectUri = OriginalPathBase + OriginalPath + Request.QueryString;
}
```
Si en vez del handler somos nosotros los que invocamos la acci�n **Challenge** de forma manual podemos establecer esta propiedad en la misma llamada:
```csharp
await HttpContext.ChallengeAsync(new AuthenticationProperties { RedirectUri = "https://myapp.com/home/wellcome" });
```

Los handlers remotos codifican las `properties` en el par�metro [**state** del est�ndar OAuth2](https://tools.ietf.org/html/rfc6749#section-4.1.1).
Se trata de un par�metro recomendado para la petici�n de autorizaci�n pero obligatorio en la respuesta si est� presente en la petici�n.
Esto significa que cuando el proveedor de identidad termine redireccionar� con todos los par�metros incluido el **state**.

Cuando el handler capture la redirecci�n decodificar� este par�metro obteniendo las propiedades originales, entre ellos la url donde deber� redireccionar la aplicaci�n para finalizar el flujo.

##### 2. La redirect uri a la que el **proveedor de identidad** debe redireccionar

Es la url formada con el path fijado en la propiedad `CallbackPath` del handler remoto y deber� coincidr exactamente con una de las *redirect uris* configuradas en el proveedor de identidad para ese cliente.

```csharp
.AddOAuth("oauth", options =>
{
    options.AuthorizationEndpoint = "https://autoridad.com/connect/authorize";
    options.TokenEndpoint = "https://autoridad.com/connect/token";
    options.ClientId = "clientmvc";
    options.ClientSecret = "secret";
    options.SignInScheme = "Cookies";
    options.CallbackPath = "/sign-in";
});
```

---

Este ser�a un ejemplo de petici�n en el endpoint de Authorization para un proveedor de identidad remoto:

```
GET https://autoridad.com/connect/authorize
        ?response_type=code
        &client_id=cliente1
        &state=U2F0IE9jdCAyNCAyMDIwIDE5OjAwOjE2IEdNVCswMjAwIChob3JhIGRlIHZlcmFubyBkZSBFdXJvcGEgY2VudHJhbCk
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

- La primera *redirect uri* vendr�a codificada en el par�metro `state` y ser�a la url a la que el usuario intent� entrar y por lo tanto la url en la que deber�a finalizar el flujo.

- La segunda *redirect uri* vendr�a en el par�metro `redirect_uri` y es la url a la que el idp deber� redireccionar para ser *capturada* por el handler remoto de nuestra aplicaci�n. Esta url adem�s deber� coincidir con una redirct uri configurada para ese cliente en el idp.

- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- Parte VI. Nota sobre redirect uri's