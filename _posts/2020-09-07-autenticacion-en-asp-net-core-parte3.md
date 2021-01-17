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
- Parte III. �C�mo se invocan las acciones?
- **[Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)**
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)

## �C�mo se invocan las acciones?
Gracias a los Handlers y al middleware de autenticaci�n nos evitamos el tener que invocar la mayor�a de las acciones.
Sin embargo, en los casos donde tengamos que invocar estas acciones nosotros mismos podremos usar uno de los siguientes m�todos:

##### Mediante la interfaz IAuthenticationService

Al invocar el m�todo `AddAuthentication(...)`, adem�s de permitirnos configurar los esquemas por defecto y devolver un builder para encadenar los m�todos de extensi�n que registran nuestros handlers, tambi�n est� registrando en el contenedor de dependencias una implementaci�n de `IAuthenticationService` por defecto.
Esta interfaz expone todas las acciones vistas anteriormente:
```csharp
Task<AuthenticateResult> AuthenticateAsync(HttpContext context, string scheme);
Task ChallengeAsync(HttpContext context, string scheme, AuthenticationProperties properties);
Task ForbidAsync(HttpContext context, string scheme, AuthenticationProperties properties);
Task SignInAsync(HttpContext context, string scheme, ClaimsPrincipal principal, AuthenticationProperties properties);
Task SignOutAsync(HttpContext context, string scheme, AuthenticationProperties properties);
```
La implementaci�n por defecto mantiene la lista de esquemas y de handlers configurados para nuestra aplicaci�n.
Dependiendo del esquema que se pase como par�metro, gracias a la relaci�n esquema-handler establecida, sabr� a que handler delegar la acci�n requerida.

Aunque podemos utilizar esta interfaz directamente lo m�s habitual es utilizar los m�todos de extensi�n de `HttpContext` o los objetos `IActionResult` que veremos a continuaci�n.
Sea cual sea el m�todo de invocaci�n utilizado ser� esta implementaci�n la que en �ltima instancia se encargar� de invocar las acciones de los handlers correspondientes.

##### Mediante los m�todos de extensi�n de HttpContext
Dentro del paquete Microsoft.AspNetCore.Authentication.Abstractions tenemos una serie de m�todos de extensi�n sobre `HttpContext` muy �tiles para la autenticaci�n.
Por un lado tenemos envolventes para los m�todos de la interfaz `IAuthenticationService` con diversas sobrecargas. Est�n son algunas de ellas:
```csharp
public static Task<AuthenticateResult> AuthenticateAsync(this HttpContext context); // +1 sobrecarga
public static Task ChallengeAsync(this HttpContext context); // +3 sobrecargas
public static Task ForbidAsync(this HttpContext context); // +3 sobrecargas
public static Task SignInAsync(this HttpContext context, ClaimsPrincipal principal); // +3 sobrecargas
public static Task SignOutAsync(this HttpContext context); // +3 sobrecargas
```
Tambi�n se expone un m�todo con dos sobrecargas para extraer el valor de un token en caso de que exista un token de autenticaci�n en la petici�n:
```csharp
public static Task<string> GetTokenAsync(this HttpContext context, string scheme, string tokenName);
public static Task<string> GetTokenAsync(this HttpContext context, string tokenName);
```

##### Mediante objetos IActionResult
Tambi�n podemos usar una serie de objetos para devolver como resultado del m�todo de acci�n.
Estos objetos se encargar�n de invocar la acci�n correspondiente durante la ejecuci�n de la petici�n usando el m�todo de extensi�n sobre HttpContext.
Todos ellos implementan `IActionResult` y cuentan con varias sobrecargas en sus constructores.

```csharp
return new ChallengeResult("oidc");                    // Invocar� la acci�n Challenge para el esquema "oidc"
return new ForbidResult();                             // Invocar� la acci�n Forbid para el esquema configurado por defecto para esta acci�n
return new SignInResult("Cookies", principal);         // Invocar� la acci�n SignIn para el esquema "Cookies" para persistir principal
return new SignOutResult(new[] { "Cookies", "oidc" }); // Invocar� la acci�n SignOut para los esquemas "Cookies" y "oidc"
```

##### Mediante m�todos de ControllerBase
Simplemente son una serie de m�todos que crean instancias de los objetos IActionResult del apartado anterior. De esta forma podemos convertir el new `[Action]Result()` en `[Action]()`.
Estos m�todos pertenecen a la clase ControllerBase por lo que pueden ser invocados desde el controlador. Al igual que los anteriores tambi�n cuentan con varias sobrecargas.
```csharp
return Challenge("oidc");              // Crear� un objeto ChallengeResult para el esquema "oidc"
return Forbid("");                     // Crear� un objeto ForbidResult para el esquema configurado por defecto para esta acci�n
return SignIn(principal, "Cookies");   // Crear� un objeto SignInResult para el esquema "Cookies" para persistir el principal
return SignOut("Cookies", "oidc");     // Crear� un objeto SignOutResult para los esquemas "Cookies" y "oidc"
```

- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- Parte III. �C�mo se invocan las acciones?
- **[Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)**
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)