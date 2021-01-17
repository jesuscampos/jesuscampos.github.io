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
- Parte I. Introducción
- **[Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)**
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)
  
# Introducción
Escribo este post para tratar de aclarar qué son y como se configuran los **esquemas de autenticación** dentro del contexto de una aplicación ASP.NET Core.
Creo que no hay demasiada documentación sobre las distintas **acciones** que combinadas forman el ciclo completo de la autenticación y de cómo se relacionan con los **esquemas**.
Esto unido a que el propio framework haga gran parte del trabajo de forma "oculta" para nosotros hace que resulte muy confusa la experiencia del desarrollador en este aspecto.

Antes de continuar quisiera definir algunos conceptos base:

- **Autenticación**: Es el proceso mediante el cual se determina la identidad de un usuario.
    Más adelante nos centraremos en especificar y configurar el método (o métodos) de autenticación utilizados dentro de nuestra aplicación.
- **Esquema de autenticación**: Es la forma en que .Net se refiere a un conjunto de *acciones* que forman un método de autenticación concreta. 
    Estas acciones, traducidas a código, vienen encapsuladas en lo que llamamos un *handler de autenticación*.
    Básicamente, a la hora de configurar la autenticación, lo que haremos será **registrar un *handler de autenticación* junto con sus opciones y asignando un *nombre de esquema* (string)** para cada método de autenticación que queramos configurar.
- **Acción de autenticación**: Cada una de las acciones que forman parte del ciclo de vida de la autenticación.
    Veremos que no todos los métodos de autenticación implementan todas las acciones.
- **Handler de autenticación**: Como base podemos decir que se trata de una implementación de `IAuthenticationHandler`, cuyo contrato compromete tres acciones de autenticación: `AuthenticateAsync`, `ChallengeAsync` y `ForbidAsync`, además de un método `InitializeAsync` que es llamado en cada petición por si necesita algún tipo de inicialización, normalmente validación de las opciones, inicialización de eventos, etc.
    Algunos handlers de autenticación cuentan con más acciones como `SignInAsync` y `SignOutAsync`. Estas acciones han sido segregadas en otras interfaces para complementar el handler base.
    Por ejemplo, el handler `CookieAuthenticationHandler` implementa además `IAuthenticationSignInHandler` y `IAuthenticationSignOutHandler` para llevar a cabo las respectivas acciones.
    Además, existe otra interfaz que implementan ciertos handlers para manejar **autenticación remota** (OAuth2, OpenId...).
    Esta interfaz es `IAuthenticationRequestHandler` e implementa el método `HandleRequestAsync` que le permitirá capturar ciertas peticiones para llevar a cabo flujos más complejos basados en proveedores de identidad externos (idp).
- **Ticket de autenticación**: Es el objeto que se produce tras una autenticación existosa.
    Contiene información de la **identidad** del usuario e información adicional del estado de autenticación.
    Parte de esta información vendrá encapsulada en forma de **claims** (o declaraciones) en un objeto `ClaimsPrincipal`.
- **ClaimsPrincipal**: ASP.NET Core [basa la autenticación en **claims**](https://en.wikipedia.org/wiki/Claims-based_identity), donde la identidad del usuario viene representada como un conjunto de claims sobre el usuario, como su nombre, su email, roles, etc. La autenticación de un usuario implica la creación de un ClaimsPrincipal que identifique al usuario en el transcurso de una petición.
    Esta clase implementa [IPrincipal](https://docs.microsoft.com/es-es/dotnet/api/system.security.principal.iprincipal?view=netcore-3.1) que representa el contexto de seguridad del usuario en cuyo nombre se ejecuta el código, incluida la identidad del usuario ([IIdentity](https://docs.microsoft.com/es-es/dotnet/api/system.security.principal.iidentity?view=netcore-3.1)) y los roles a los que pertenece.
> La **identidad** tras un IPrincipal puede ser tanto de un **usuario** como de una máquina, dependiendo en nombre de quién se está ejecutando el código.

### Acciones:

Cualquier acción que tenga que ver con el ciclo de vida de la autenticación de usuarios debe estar vinculada a un esquema.
Este esquema determinará la forma en que se lleva a cabo esta acción.

Una misma aplicación puede configurarse con varios tipos de esquemas, es decir, es posible tener en una misma aplicación distintos métodos de autenticación.
Tal y como se ha comentado anteriormente, los esquemas no tienen por qué implementar todas y cada una de las acciones.
Por estas razones es muy habitual combinar varios esquemas en la autenticación de una aplicación.
Más adelante veremos como combinar un esquema de **Cookies** y un esquema **OpenId** para definir la autenticación de una aplicación web.

A continuación se describen las distintas acciones que forman parte el ciclo de vida de la autenticación:

- **Authenticate**: Cuando una aplicación autentica a un usuario quiere decir que crea un objeto **AuthenticateResult** a partir de cierta información (Tokens, Cookies...).
  Una autenticación exitosa contendrá en este resultado un **AuthenticationTicket** con la toda esta información.
  El propio framework, a través del middleware de autenticación, se encargará de exponer esta información encapsulada en un **ClaimsPrincipal** en el propio contexto de la petición para hacerlo accesible a todo el pipeline.
- **Forbid**: Se encarga de informar al usuario que no tiene acceso a un recurso.
  La forma de responder dependerá del tipo de handler utilizado para ese esquema.
  Esta acción puede ir desde preparar una respuesta 403 Forbidden para una API protegida con un esquema basado en tokens jwt **Bearer**, hasta redirigir a una página de *Acceso denegado* para un esquema basado en **Cookies** para una aplicación MVC.
- **Challenge**: Inicia el proceso de autenticación establecido para el esquema.
  Es la acción que se invocaría por ejemplo al hacer clic en un botón *Login* donde dependamos de un idp externo, o tras un intento de acceso a un recurso protegido para forzar al usuario a realizar el proceso de login.
  Sin embargo, para una API protegida con un esquema **Bearer** simplemente preparará una respuesta [401 Unauthorized con una cabecera WWW-Authenticate](https://tools.ietf.org/html/rfc7235#section-4.1) indicando el esquema por defecto configurado para la acción Challenge.
- **Sign-in**: Esta acción le pide al framework que persista un **ClaimsPrincipal** con información del usuario actual de la forma que indique su esquema.
  Esta acción es útil para mantener la identidad del usuario entre peticiones (sesión).
  Un esquema basado en **Cookies** crearía una cookie de sesión cifrada y la enviaría al explorador para que, a partir de ese momento, todas las peticiones que haga ese usuario sean peticiones autenticadas.
  Sin embargo, en un esquema **Bearer** el cliente deberá conseguir un token previamente y deberá enviarlo en cada petición (normalmente en un header). Este token porta toda la información necesaria, por lo que no necesita implementar esta acción. De hecho, este handler no implementa ni SignIn ni SignOut.
- **Sign-out**: Realiza la acción opuesta a la anterior, es decir, le pide al esquema configurado para tal fin que elimine la persistencia del usuario.
  Por ejemplo, para el esquema basado en **Cookies** y tras un clic en el botón de *Logout*, devolveríamos al explorador la cookie de sesión con una fecha en el pasado (esta es la forma de pedirle al explorador que elimine una cookie).

A lo largo del post veremos ciertos tipos de handlers y combinaciones de ellos utilizados en distintos escenarios.
Estos son los distintos métodos de autenticación que veremos, sus handlers, y el nombre de los esquemas que usaremos para cada unos de los escenarios:

|Método/s de autenticación|Tipo de Aplicación|[^1]Nombre/s esquema/s|Handler|
|-------------|-------------|-----|
|Local mediante Cookies|Aplicación Web MVC|Cookies|CookieAuthenticationHandler|
|Remota OAuth2 + Cookies|Applicación Web MVC|Cookies y oauth|CookieAuthenticationHandler y OAuthAuthenticationHandler|
|Remota OpenId + Cookies|Applicación Web MVC|Cookies y oidc|CookieAuthenticationHandler y OpendIdAuthenticationHandler|
|Tokens tipo Jwt|Api|Bearer|JwtBearerHandler|
|Tokens tipo Reference|Api|Bearer|OAuth2IntrospectionHandler|
|Tokens tipo Jwt + Reference|Api|token e introspection|JwtBearerHandler y OAuth2IntrospectionHandler|
[^1]: Aunque el nombre que asignamos a los esquemas es arbitrario nosotros usaremos los más utilizados en la documentación existente.
Además, para nuestra comodidad, contamos en el framework con constantes para no tener que escribirlas:
`CookieAuthenticationDefaults.AuthenticationScheme = "Cookies"`; `JwtBearerDefaults.AuthenticationScheme = "Bearer"`; `OpenIdConnectDefaults.AuthenticationScheme = "OpenIdConnect"`, etc.

---------------------
#### Enlaces de interés:
Si tienes curiosidad te animo a que eches un vistazo al código fuente de algunas de las clases que vamos a ver:

- [Middleware (AuthenticationMiddleware)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationMiddleware.cs)
- [Handler base (AuthenticationHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationHandler.cs)
- [Handler remoto base (RemoteAuthenticationHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/RemoteAuthenticationHandler.cs)
- [Implementación de Handler remoto para OAuth2 (OAuthHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.OAuth/OAuthHandler.cs)
- [Implementación de Handler remoto para OpenId (OpenIdConnectHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.OpenIdConnect/OpenIdConnectHandler.cs)
- [Implementación de Handler remoto para Google (GoogleHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.Google/GoogleHandler.cs)
- [Handler Cookies (CookieAuthenticationHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.Cookies/CookieAuthenticationHandler.cs)
- [Handler Jwt Bearer (JwtBearer)](https://github.com/aspnet/Security/tree/master/src/Microsoft.AspNetCore.Authentication.JwtBearer)
- [Handler Jwt Bearer para tokens de tipo referencia (OAuth2IntrospectionHandler)](https://github.com/IdentityModel/IdentityModel.AspNetCore.OAuth2Introspection/blob/main/src/OAuth2IntrospectionHandler.cs)

----------------------
- Parte I. Introducción
- **[Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)**
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md) 