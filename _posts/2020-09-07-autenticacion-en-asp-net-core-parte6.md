---
layout: post
title: Autenticación en ASP.NET Core
author: Jesús
tags:
- ASP.NET
- Schemes
- Esquemas
- ASP.NET Core
- Startup
- C#
- Autenticación
- Authentication handler
- Middleware
- Net Core
---
- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- Parte VI. Nota sobre redirect uri's

## Nota sobre *redirect uri's*
En una autenticación remota existen dos *redirect uris* que no debemos confundir:

##### 1. La redirect uri a la que la **aplicación cliente** debe redireccionar tras un login exitoso
Suele ser la url a la que el usuario intentó entrar y la razón por la que el **Challenge** se desencadenó.

El middleware establece en el pipeline información sobre la llamada original en una de sus [Features](https://docs.microsoft.com/es-es/aspnet/core/fundamentals/request-features?view=aspnetcore-3.1) para que esté disponible en el handler.
```csharp
// AuthenticationMiddleware
context.Features.Set<IAuthenticationFeature>(new AuthenticationFeature
{
    OriginalPath = context.Request.Path,
    OriginalPathBase = context.Request.PathBase
});
```
Más adelante esta información es utilizada por el handler para crear la **propiedad de autenticación** `RedirectUri`:
```csharp
// AuthenticationHandler (base de todos los handlers)
protected PathString OriginalPath => Context.Features.Get<IAuthenticationFeature>()?.OriginalPath ?? Request.Path;
protected PathString OriginalPathBase => Context.Features.Get<IAuthenticationFeature>()?.OriginalPathBase ?? Request.PathBase;

// Por ejemplo OAuthHandler, aunque otros handlers tanto remotos como no remotos se comportarán igual
if (string.IsNullOrEmpty(properties.RedirectUri))
{
    properties.RedirectUri = OriginalPathBase + OriginalPath + Request.QueryString;
}
```
Si en vez del handler somos nosotros los que invocamos la acción **Challenge** de forma manual podemos establecer esta propiedad en la misma llamada:
```csharp
await HttpContext.ChallengeAsync(new AuthenticationProperties { RedirectUri = "https://myapp.com/home/wellcome" });
```

Los handlers remotos codifican las `properties` en el parámetro [**state** del estándar OAuth2](https://tools.ietf.org/html/rfc6749#section-4.1.1).
Se trata de un parámetro recomendado para la petición de autorización pero obligatorio en la respuesta si está presente en la petición.
Esto significa que cuando el proveedor de identidad termine redireccionará con todos los parámetros incluido el **state**.

Cuando el handler capture la redirección decodificará este parámetro obteniendo las propiedades originales, entre ellos la url donde deberá redireccionar la aplicación para finalizar el flujo.

##### 2. La redirect uri a la que el **proveedor de identidad** debe redireccionar

Es la url formada con el path fijado en la propiedad `CallbackPath` del handler remoto y deberá coincidr exactamente con una de las *redirect uris* configuradas en el proveedor de identidad para ese cliente.

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

Este sería un ejemplo de petición en el endpoint de Authorization para un proveedor de identidad remoto:

```
GET https://autoridad.com/connect/authorize
        ?response_type=code
        &client_id=cliente1
        &state=U2F0IE9jdCAyNCAyMDIwIDE5OjAwOjE2IEdNVCswMjAwIChob3JhIGRlIHZlcmFubyBkZSBFdXJvcGEgY2VudHJhbCk
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

- La primera *redirect uri* vendría codificada en el parámetro `state` y sería la url a la que el usuario intentó entrar y por lo tanto la url en la que debería finalizar el flujo.

- La segunda *redirect uri* vendría en el parámetro `redirect_uri` y es la url a la que el idp deberá redireccionar para ser *capturada* por el handler remoto de nuestra aplicación. Esta url además deberá coincidir con una redirct uri configurada para ese cliente en el idp.

- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- Parte VI. Nota sobre redirect uri's