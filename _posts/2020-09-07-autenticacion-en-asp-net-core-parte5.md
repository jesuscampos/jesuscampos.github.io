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
- Authentication
- Remote authentication
- Autenticación remota
- Authentication handler
- Middleware de autenticación
- Authentication Middleware
- Net Core
---
- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- Parte V. Middleware de autenticación
- **[Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)**

## Middleware de autenticación
La autenticación de cada petición siempre se incia en el [middleware de autenticación](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationMiddleware.cs) y su funcionamiento es común para todos los esquemas.

Insertado en el pipeline de nuestro servicio, es el encargado de *intentar* la autenticación del usario/cliente y lo hará independientemente del tipo de esquema utilizado para todas las peticiones que lleguen al middleware.

A priori, una petición entrante es anónima hasta que no se produzca la autenticación. El middleware se encargará de intentar averiguar si hay un usuario detrás de cada petición.

Para el middleware existen tres situaciones posibles en el momento en que recibe la petición:

##### Caso I. El servidor de identidad externo a redireccionado a nuestra aplicación tras un login

Sólo es posible si nuestra aplicación cuenta con **autenticación remota** (OpenId Connect, OAuth2...).
En este caso la información vendrá en forma de **tokens** o de un **code** para intercambiarlo, según el [tipo de respuesta](https://tools.ietf.org/html/rfc6749#section-3.1.1) utilizado en el flujo.
El middleware extraerá la información contenida en la propia petición.

>Para llegar a esta situación debió haberse producido antes un **Challenge** en nuestra aplicación.
Esta acción hizo que el usuario fuera redirigido a la página de login del proveedor de identidad externo.
Tras un login exitoso el proveedor volvió a redirigir al usuario a la aplicación produciéndose esta situación.

##### Caso II. El usuario ya está autenticado
Las peticiones vendrán con información sobre la autenticación. Esta información podrá venir de diversas maneras: una **cookie de sesión**, un token en una cabecera, en el cuerpo del mensaje, etc.
El middleware deberá autenticar al usuario extrayendo esta información para construir una identidad y exponerla para el pipeline.

##### Caso III. No hay usuario
Para el resto de los casos la petición es anónima ya que no será posible su autenticación.

### Proceso de autenticación de una petición
El siguiente diagrama muestra el funcionamiento básico del middleware y de como intenta realizar la autenticación, primero de forma remota, y si no lo consigue así, lo intentará usando el esquema por defecto de la aplicación.

![AuthenticationMiddleware.png]({{site.baseurl}}/images/authentication-middleware.png)

#### Primer intento: Autenticación remota
**Es la autenticación que se produciría tras una redirección de login desde un proveedor de identidad externo.** (Caso I).

Lo primero que hace el middleware es recorrer los handlers remotos que hayamos configurado en la aplicación (implementaciones de `IAuthenticationHandler`).
Si tenemos alguno configurado llamará al método `HandleRequestAsync` para darle la oportunidad de capturar la petición para que se produzca una **autenticación remota**.

El handler remoto decidirá si es o no para él la petición comparando el path de la petición con el path configurado en las opciones del handler remoto: `CallbackPath`.
Por ejemplo, en el siguiente código configuraremos el callback de sign-in para un handler de tipo OAuth2:

```csharp
// Combinamos Cookies + oauth
services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies";
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOAuth("oauth", options =>
{
    options.AuthorizationEndpoint = "https://autoridad.com/account/authorize";
    options.TokenEndpoint = "https://autoridad.com/account/token";
    options.ClientId = "clientmvc";
    options.ClientSecret = "secret";
    options.SignInScheme = "Cookies";
    options.CallbackPath = "/sign-in";
});
```

> Hay que tener en cuenta que para flujos OAuth2 y OpenId Connect, este callback formará parte del parámetro redirectUri al inciarse el flujo de login remoto y que deberá coincidir la url completa con una de las *redirect uris* configuradas para el cliente.

En el caso de que el path sea capturado por el handle remoto, el propio handler se encargará de invocar su método abstracto `HandleRemoteAuthenticateAsync`.
La implementación de este método varía en función del tipo concreto de handler. Ejemplos:
- Para un handler OAuth2 obtendrá el `code` de la query y lo intercambiará por un AccessToken en el **TokenEndpoint**. Con este token creará la identidad del usuario.
- Para un handler OpenId Connect obtendrá los tokens de la query si es un get, o del body si es un post. Dependiendo de los tipos de tokens recibidos obtendrá la información del usuario de distintas formas:
  a partir de un IdToken, de un Code intercambiando en TokenEndpoint por un AccessToken, usando un AccessToken directamente o incluso realizando una llamada al endpoint **UserInfo** si así estuviera configurado.

Sea cual sea el método habremos obtenido una identidad del usuario en forma de `AuthenticationTicket`.
El propio handler utilizará el `ClaimsPrincipal` contenido en este ticket para invocar la acción **SignIn**.
Para esta invocación utilizará el esquema configurado en la propiedad `SignInScheme`.

>Aunque en el ejemplo hemos asignado `"Cookies"` en esta propiedad de forma explícita no hubiera sido necesario, ya que si no se especifica esta propiedad hubiera utilizado el esquema por defecto que es precisamente "Cookies".

Una vez autenticado de forma remota el usuario y persistida la sesión en una cookie, el handler procederá a preparar una respuesta de redirección a la url de destino final.
Esta respuesta envía la cookie al cliente, por lo que en sucesivas peticiones (incluida la propia redirección) ya vendrá autenticada desde el cliente (serán peticiones de Caso II).

>La url de destino final puede ser la url a la que el usuario intentó entrar cuando se "disparó" el **Challenge** o símplemente la url raiz "/" si fue un **Challenge** manual.

Cuando se produzca una autenticación remota el middleware de autenticación no invocará el delegado `next()` del middleware, si no que retornará convirtiéndose así en el último middleware del pipeline.

Este es el código del middleware donde realiza los intentos de autenticación remota. Puede verse que en caso de éxito será el último middleware del pipeline en ejecutarse.

```csharp
// Give any IAuthenticationRequestHandler schemes a chance to handle the request
var handlers = context.RequestServices.GetRequiredService<IAuthenticationHandlerProvider>();
foreach (var scheme in await Schemes.GetRequestHandlerSchemesAsync())
{
    var handler = await handlers.GetHandlerAsync(context, scheme.Name) as IAuthenticationRequestHandler;
    if (handler != null && await handler.HandleRequestAsync())
    {
        return;        // Retorna al middleware anterior
    }
}
```

Si no se produjo autenticación remota continuará el middleware para intentar autenticar la petición con el esquema por defecto local.

#### Segundo intento: Autenticación local
**Es la autenticación que se produciría por defecto en la aplicación si no se captura antes por una autenticación remota.** (Caso I).

El middleware usará el esquema por defecto para la acción **Authenticate** para intentar extraer la información del usuario llamando al método de extensión `AuthenticateAsync` pasando como parámetro el esquema por defecto para esta acción.

Para el caso de un esquema "Cookies" nuestro handler intentará leer la cookie de sesión. Si existe podrá extraer toda la información.

Para el caso de un token "Bearer" el handler intentará obtener la información del token que venga en la petición (normalmente en el header Authorization).

Sea cual sea el handler utilizado, si se ha producido autenticación, nos devolverá un `AuthenticationTicket` con la información necesaria contenida en un `ClaimsPrincipal`.
Esta es la parte de código del middleware donde asigna la identidad conseguida `result.Principal` a la propiedad `User` del `HttpContext` para que esté disponible a lo largo del pipeline.

```csharp
var defaultAuthenticate = await Schemes.GetDefaultAuthenticateSchemeAsync();
if (defaultAuthenticate != null)
{
    var result = await context.AuthenticateAsync(defaultAuthenticate.Name);
    if (result?.Principal != null)
    {
        context.User = result.Principal;
    }
}

await _next(context);
```

Sin embargo, si no se produce autenticación la petición quedará como anónima ya que la propiedad `User` será nula (caso III).

Aquí terminará el middleware su trabajo pasando la ejecución al siguiente middleware invocando el delegado `_next`.

- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- Parte V. Middleware de autenticación
- **[Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)**