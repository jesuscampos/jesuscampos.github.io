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
- Parte I. Introducci�n
- **[Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)**
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)
  
# Introducci�n
Escribo este post para tratar de aclarar qu� son y como se configuran los **esquemas de autenticaci�n** dentro del contexto de una aplicaci�n ASP.NET Core.
Creo que no hay demasiada documentaci�n sobre las distintas **acciones** que combinadas forman el ciclo completo de la autenticaci�n y de c�mo se relacionan con los **esquemas**.
Esto unido a que el propio framework haga gran parte del trabajo de forma "oculta" para nosotros hace que resulte muy confusa la experiencia del desarrollador en este aspecto.

Antes de continuar quisiera definir algunos conceptos base:

- **Autenticaci�n**: Es el proceso mediante el cual se determina la identidad de un usuario.
    M�s adelante nos centraremos en especificar y configurar el m�todo (o m�todos) de autenticaci�n utilizados dentro de nuestra aplicaci�n.
- **Esquema de autenticaci�n**: Es la forma en que .Net se refiere a un conjunto de *acciones* que forman un m�todo de autenticaci�n concreta. 
    Estas acciones, traducidas a c�digo, vienen encapsuladas en lo que llamamos un *handler de autenticaci�n*.
    B�sicamente, a la hora de configurar la autenticaci�n, lo que haremos ser� **registrar un *handler de autenticaci�n* junto con sus opciones y asignando un *nombre de esquema* (string)** para cada m�todo de autenticaci�n que queramos configurar.
- **Acci�n de autenticaci�n**: Cada una de las acciones que forman parte del ciclo de vida de la autenticaci�n.
    Veremos que no todos los m�todos de autenticaci�n implementan todas las acciones.
- **Handler de autenticaci�n**: Como base podemos decir que se trata de una implementaci�n de `IAuthenticationHandler`, cuyo contrato compromete tres acciones de autenticaci�n: `AuthenticateAsync`, `ChallengeAsync` y `ForbidAsync`, adem�s de un m�todo `InitializeAsync` que es llamado en cada petici�n por si necesita alg�n tipo de inicializaci�n, normalmente validaci�n de las opciones, inicializaci�n de eventos, etc.
    Algunos handlers de autenticaci�n cuentan con m�s acciones como `SignInAsync` y `SignOutAsync`. Estas acciones han sido segregadas en otras interfaces para complementar el handler base.
    Por ejemplo, el handler `CookieAuthenticationHandler` implementa adem�s `IAuthenticationSignInHandler` y `IAuthenticationSignOutHandler` para llevar a cabo las respectivas acciones.
    Adem�s, existe otra interfaz que implementan ciertos handlers para manejar **autenticaci�n remota** (OAuth2, OpenId...).
    Esta interfaz es `IAuthenticationRequestHandler` e implementa el m�todo `HandleRequestAsync` que le permitir� capturar ciertas peticiones para llevar a cabo flujos m�s complejos basados en proveedores de identidad externos (idp).
- **Ticket de autenticaci�n**: Es el objeto que se produce tras una autenticaci�n existosa.
    Contiene informaci�n de la **identidad** del usuario e informaci�n adicional del estado de autenticaci�n.
    Parte de esta informaci�n vendr� encapsulada en forma de **claims** (o declaraciones) en un objeto `ClaimsPrincipal`.
- **ClaimsPrincipal**: ASP.NET Core [basa la autenticaci�n en **claims**](https://en.wikipedia.org/wiki/Claims-based_identity), donde la identidad del usuario viene representada como un conjunto de claims sobre el usuario, como su nombre, su email, roles, etc. La autenticaci�n de un usuario implica la creaci�n de un ClaimsPrincipal que identifique al usuario en el transcurso de una petici�n.
    Esta clase implementa [IPrincipal](https://docs.microsoft.com/es-es/dotnet/api/system.security.principal.iprincipal?view=netcore-3.1) que representa el contexto de seguridad del usuario en cuyo nombre se ejecuta el c�digo, incluida la identidad del usuario ([IIdentity](https://docs.microsoft.com/es-es/dotnet/api/system.security.principal.iidentity?view=netcore-3.1)) y los roles a los que pertenece.
> La **identidad** tras un IPrincipal puede ser tanto de un **usuario** como de una m�quina, dependiendo en nombre de qui�n se est� ejecutando el c�digo.

### Acciones:

Cualquier acci�n que tenga que ver con el ciclo de vida de la autenticaci�n de usuarios debe estar vinculada a un esquema.
Este esquema determinar� la forma en que se lleva a cabo esta acci�n.

Una misma aplicaci�n puede configurarse con varios tipos de esquemas, es decir, es posible tener en una misma aplicaci�n distintos m�todos de autenticaci�n.
Tal y como se ha comentado anteriormente, los esquemas no tienen por qu� implementar todas y cada una de las acciones.
Por estas razones es muy habitual combinar varios esquemas en la autenticaci�n de una aplicaci�n.
M�s adelante veremos como combinar un esquema de **Cookies** y un esquema **OpenId** para definir la autenticaci�n de una aplicaci�n web.

A continuaci�n se describen las distintas acciones que forman parte el ciclo de vida de la autenticaci�n:

- **Authenticate**: Cuando una aplicaci�n autentica a un usuario quiere decir que crea un objeto **AuthenticateResult** a partir de cierta informaci�n (Tokens, Cookies...).
  Una autenticaci�n exitosa contendr� en este resultado un **AuthenticationTicket** con la toda esta informaci�n.
  El propio framework, a trav�s del middleware de autenticaci�n, se encargar� de exponer esta informaci�n encapsulada en un **ClaimsPrincipal** en el propio contexto de la petici�n para hacerlo accesible a todo el pipeline.
- **Forbid**: Se encarga de informar al usuario que no tiene acceso a un recurso.
  La forma de responder depender� del tipo de handler utilizado para ese esquema.
  Esta acci�n puede ir desde preparar una respuesta 403 Forbidden para una API protegida con un esquema basado en tokens jwt **Bearer**, hasta redirigir a una p�gina de *Acceso denegado* para un esquema basado en **Cookies** para una aplicaci�n MVC.
- **Challenge**: Inicia el proceso de autenticaci�n establecido para el esquema.
  Es la acci�n que se invocar�a por ejemplo al hacer clic en un bot�n *Login* donde dependamos de un idp externo, o tras un intento de acceso a un recurso protegido para forzar al usuario a realizar el proceso de login.
  Sin embargo, para una API protegida con un esquema **Bearer** simplemente preparar� una respuesta [401 Unauthorized con una cabecera WWW-Authenticate](https://tools.ietf.org/html/rfc7235#section-4.1) indicando el esquema por defecto configurado para la acci�n Challenge.
- **Sign-in**: Esta acci�n le pide al framework que persista un **ClaimsPrincipal** con informaci�n del usuario actual de la forma que indique su esquema.
  Esta acci�n es �til para mantener la identidad del usuario entre peticiones (sesi�n).
  Un esquema basado en **Cookies** crear�a una cookie de sesi�n cifrada y la enviar�a al explorador para que, a partir de ese momento, todas las peticiones que haga ese usuario sean peticiones autenticadas.
  Sin embargo, en un esquema **Bearer** el cliente deber� conseguir un token previamente y deber� enviarlo en cada petici�n (normalmente en un header). Este token porta toda la informaci�n necesaria, por lo que no necesita implementar esta acci�n. De hecho, este handler no implementa ni SignIn ni SignOut.
- **Sign-out**: Realiza la acci�n opuesta a la anterior, es decir, le pide al esquema configurado para tal fin que elimine la persistencia del usuario.
  Por ejemplo, para el esquema basado en **Cookies** y tras un clic en el bot�n de *Logout*, devolver�amos al explorador la cookie de sesi�n con una fecha en el pasado (esta es la forma de pedirle al explorador que elimine una cookie).

A lo largo del post veremos ciertos tipos de handlers y combinaciones de ellos utilizados en distintos escenarios.
Estos son los distintos m�todos de autenticaci�n que veremos, sus handlers, y el nombre de los esquemas que usaremos para cada unos de los escenarios:

|M�todo/s de autenticaci�n|Tipo de Aplicaci�n|[^1]Nombre/s esquema/s|Handler|
|-------------|-------------|-----|
|Local mediante Cookies|Aplicaci�n Web MVC|Cookies|CookieAuthenticationHandler|
|Remota OAuth2 + Cookies|Applicaci�n Web MVC|Cookies y oauth|CookieAuthenticationHandler y OAuthAuthenticationHandler|
|Remota OpenId + Cookies|Applicaci�n Web MVC|Cookies y oidc|CookieAuthenticationHandler y OpendIdAuthenticationHandler|
|Tokens tipo Jwt|Api|Bearer|JwtBearerHandler|
|Tokens tipo Reference|Api|Bearer|OAuth2IntrospectionHandler|
|Tokens tipo Jwt + Reference|Api|token e introspection|JwtBearerHandler y OAuth2IntrospectionHandler|
[^1]: Aunque el nombre que asignamos a los esquemas es arbitrario nosotros usaremos los m�s utilizados en la documentaci�n existente.
Adem�s, para nuestra comodidad, contamos en el framework con constantes para no tener que escribirlas:
`CookieAuthenticationDefaults.AuthenticationScheme = "Cookies"`; `JwtBearerDefaults.AuthenticationScheme = "Bearer"`; `OpenIdConnectDefaults.AuthenticationScheme = "OpenIdConnect"`, etc.

---------------------
#### Enlaces de inter�s:
Si tienes curiosidad te animo a que eches un vistazo al c�digo fuente de algunas de las clases que vamos a ver:

- [Middleware (AuthenticationMiddleware)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationMiddleware.cs)
- [Handler base (AuthenticationHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/AuthenticationHandler.cs)
- [Handler remoto base (RemoteAuthenticationHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication/RemoteAuthenticationHandler.cs)
- [Implementaci�n de Handler remoto para OAuth2 (OAuthHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.OAuth/OAuthHandler.cs)
- [Implementaci�n de Handler remoto para OpenId (OpenIdConnectHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.OpenIdConnect/OpenIdConnectHandler.cs)
- [Implementaci�n de Handler remoto para Google (GoogleHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.Google/GoogleHandler.cs)
- [Handler Cookies (CookieAuthenticationHandler)](https://github.com/aspnet/Security/blob/master/src/Microsoft.AspNetCore.Authentication.Cookies/CookieAuthenticationHandler.cs)
- [Handler Jwt Bearer (JwtBearer)](https://github.com/aspnet/Security/tree/master/src/Microsoft.AspNetCore.Authentication.JwtBearer)
- [Handler Jwt Bearer para tokens de tipo referencia (OAuth2IntrospectionHandler)](https://github.com/IdentityModel/IdentityModel.AspNetCore.OAuth2Introspection/blob/main/src/OAuth2IntrospectionHandler.cs)

----------------------
- Parte I. Introducci�n
- **[Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)**
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md) 