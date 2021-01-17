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
- Authentication
- Remote authentication
- Autenticaci�n remota
- Authentication handler
- Middleware de autenticaci�n
- Authentication Middleware
- Net Core
---
- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- Parte V. Middleware de autenticaci�n
- **[Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)**

## Middleware de autenticaci�n
La autenticaci�n de cada petici�n siempre se incia en el [middleware de autenticaci�n](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationMiddleware.cs) y su funcionamiento es com�n para todos los esquemas.

Insertado en el pipeline de nuestro servicio, es el encargado de *intentar* la autenticaci�n del usario/cliente y lo har� independientemente del tipo de esquema utilizado para todas las peticiones que lleguen al middleware.

A priori, una petici�n entrante es an�nima hasta que no se produzca la autenticaci�n. El middleware se encargar� de intentar averiguar si hay un usuario detr�s de cada petici�n.

Para el middleware existen tres situaciones posibles en el momento en que recibe la petici�n:

##### Caso I. El servidor de identidad externo a redireccionado a nuestra aplicaci�n tras un login

S�lo es posible si nuestra aplicaci�n cuenta con **autenticaci�n remota** (OpenId Connect, OAuth2...).
En este caso la informaci�n vendr� en forma de **tokens** o de un **code** para intercambiarlo, seg�n el [tipo de respuesta](https://tools.ietf.org/html/rfc6749#section-3.1.1) utilizado en el flujo.
El middleware extraer� la informaci�n contenida en la propia petici�n.

>Para llegar a esta situaci�n debi� haberse producido antes un **Challenge** en nuestra aplicaci�n.
Esta acci�n hizo que el usuario fuera redirigido a la p�gina de login del proveedor de identidad externo.
Tras un login exitoso el proveedor volvi� a redirigir al usuario a la aplicaci�n produci�ndose esta situaci�n.

##### Caso II. El usuario ya est� autenticado
Las peticiones vendr�n con informaci�n sobre la autenticaci�n. Esta informaci�n podr� venir de diversas maneras: una **cookie de sesi�n**, un token en una cabecera, en el cuerpo del mensaje, etc.
El middleware deber� autenticar al usuario extrayendo esta informaci�n para construir una identidad y exponerla para el pipeline.

##### Caso III. No hay usuario
Para el resto de los casos la petici�n es an�nima ya que no ser� posible su autenticaci�n.

### Proceso de autenticaci�n de una petici�n
El siguiente diagrama muestra el funcionamiento b�sico del middleware y de como intenta realizar la autenticaci�n, primero de forma remota, y si no lo consigue as�, lo intentar� usando el esquema por defecto de la aplicaci�n.

![AuthenticationMiddleware.png]({{site.baseurl}}/images/authentication-middleware.png)

#### Primer intento: Autenticaci�n remota
**Es la autenticaci�n que se producir�a tras una redirecci�n de login desde un proveedor de identidad externo.** (Caso I).

Lo primero que hace el middleware es recorrer los handlers remotos que hayamos configurado en la aplicaci�n (implementaciones de `IAuthenticationHandler`).
Si tenemos alguno configurado llamar� al m�todo `HandleRequestAsync` para darle la oportunidad de capturar la petici�n para que se produzca una **autenticaci�n remota**.

El handler remoto decidir� si es o no para �l la petici�n comparando el path de la petici�n con el path configurado en las opciones del handler remoto: `CallbackPath`.
Por ejemplo, en el siguiente c�digo configuraremos el callback de sign-in para un handler de tipo OAuth2:

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

> Hay que tener en cuenta que para flujos OAuth2 y OpenId Connect, este callback formar� parte del par�metro redirectUri al inciarse el flujo de login remoto y que deber� coincidir la url completa con una de las *redirect uris* configuradas para el cliente.

En el caso de que el path sea capturado por el handle remoto, el propio handler se encargar� de invocar su m�todo abstracto `HandleRemoteAuthenticateAsync`.
La implementaci�n de este m�todo var�a en funci�n del tipo concreto de handler. Ejemplos:
- Para un handler OAuth2 obtendr� el `code` de la query y lo intercambiar� por un AccessToken en el **TokenEndpoint**. Con este token crear� la identidad del usuario.
- Para un handler OpenId Connect obtendr� los tokens de la query si es un get, o del body si es un post. Dependiendo de los tipos de tokens recibidos obtendr� la informaci�n del usuario de distintas formas:
  a partir de un IdToken, de un Code intercambiando en TokenEndpoint por un AccessToken, usando un AccessToken directamente o incluso realizando una llamada al endpoint **UserInfo** si as� estuviera configurado.

Sea cual sea el m�todo habremos obtenido una identidad del usuario en forma de `AuthenticationTicket`.
El propio handler utilizar� el `ClaimsPrincipal` contenido en este ticket para invocar la acci�n **SignIn**.
Para esta invocaci�n utilizar� el esquema configurado en la propiedad `SignInScheme`.

>Aunque en el ejemplo hemos asignado `"Cookies"` en esta propiedad de forma expl�cita no hubiera sido necesario, ya que si no se especifica esta propiedad hubiera utilizado el esquema por defecto que es precisamente "Cookies".

Una vez autenticado de forma remota el usuario y persistida la sesi�n en una cookie, el handler proceder� a preparar una respuesta de redirecci�n a la url de destino final.
Esta respuesta env�a la cookie al cliente, por lo que en sucesivas peticiones (incluida la propia redirecci�n) ya vendr� autenticada desde el cliente (ser�n peticiones de Caso II).

>La url de destino final puede ser la url a la que el usuario intent� entrar cuando se "dispar�" el **Challenge** o s�mplemente la url raiz "/" si fue un **Challenge** manual.

Cuando se produzca una autenticaci�n remota el middleware de autenticaci�n no invocar� el delegado `next()` del middleware, si no que retornar� convirti�ndose as� en el �ltimo middleware del pipeline.

Este es el c�digo del middleware donde realiza los intentos de autenticaci�n remota. Puede verse que en caso de �xito ser� el �ltimo middleware del pipeline en ejecutarse.

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

Si no se produjo autenticaci�n remota continuar� el middleware para intentar autenticar la petici�n con el esquema por defecto local.

#### Segundo intento: Autenticaci�n local
**Es la autenticaci�n que se producir�a por defecto en la aplicaci�n si no se captura antes por una autenticaci�n remota.** (Caso I).

El middleware usar� el esquema por defecto para la acci�n **Authenticate** para intentar extraer la informaci�n del usuario llamando al m�todo de extensi�n `AuthenticateAsync` pasando como par�metro el esquema por defecto para esta acci�n.

Para el caso de un esquema "Cookies" nuestro handler intentar� leer la cookie de sesi�n. Si existe podr� extraer toda la informaci�n.

Para el caso de un token "Bearer" el handler intentar� obtener la informaci�n del token que venga en la petici�n (normalmente en el header Authorization).

Sea cual sea el handler utilizado, si se ha producido autenticaci�n, nos devolver� un `AuthenticationTicket` con la informaci�n necesaria contenida en un `ClaimsPrincipal`.
Esta es la parte de c�digo del middleware donde asigna la identidad conseguida `result.Principal` a la propiedad `User` del `HttpContext` para que est� disponible a lo largo del pipeline.

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

Sin embargo, si no se produce autenticaci�n la petici�n quedar� como an�nima ya que la propiedad `User` ser� nula (caso III).

Aqu� terminar� el middleware su trabajo pasando la ejecuci�n al siguiente middleware invocando el delegado `_next`.

- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- Parte V. Middleware de autenticaci�n
- **[Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)**