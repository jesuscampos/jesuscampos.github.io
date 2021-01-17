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
- Parte III. ¿Cómo se invocan las acciones?
- **[Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)**
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)

## ¿Cómo se invocan las acciones?
Gracias a los Handlers y al middleware de autenticación nos evitamos el tener que invocar la mayoría de las acciones.
Sin embargo, en los casos donde tengamos que invocar estas acciones nosotros mismos podremos usar uno de los siguientes métodos:

##### Mediante la interfaz IAuthenticationService

Al invocar el método `AddAuthentication(...)`, además de permitirnos configurar los esquemas por defecto y devolver un builder para encadenar los métodos de extensión que registran nuestros handlers, también está registrando en el contenedor de dependencias una implementación de `IAuthenticationService` por defecto.
Esta interfaz expone todas las acciones vistas anteriormente:
```csharp
Task<AuthenticateResult> AuthenticateAsync(HttpContext context, string scheme);
Task ChallengeAsync(HttpContext context, string scheme, AuthenticationProperties properties);
Task ForbidAsync(HttpContext context, string scheme, AuthenticationProperties properties);
Task SignInAsync(HttpContext context, string scheme, ClaimsPrincipal principal, AuthenticationProperties properties);
Task SignOutAsync(HttpContext context, string scheme, AuthenticationProperties properties);
```
La implementación por defecto mantiene la lista de esquemas y de handlers configurados para nuestra aplicación.
Dependiendo del esquema que se pase como parámetro, gracias a la relación esquema-handler establecida, sabrá a que handler delegar la acción requerida.

Aunque podemos utilizar esta interfaz directamente lo más habitual es utilizar los métodos de extensión de `HttpContext` o los objetos `IActionResult` que veremos a continuación.
Sea cual sea el método de invocación utilizado será esta implementación la que en última instancia se encargará de invocar las acciones de los handlers correspondientes.

##### Mediante los métodos de extensión de HttpContext
Dentro del paquete Microsoft.AspNetCore.Authentication.Abstractions tenemos una serie de métodos de extensión sobre `HttpContext` muy útiles para la autenticación.
Por un lado tenemos envolventes para los métodos de la interfaz `IAuthenticationService` con diversas sobrecargas. Están son algunas de ellas:
```csharp
public static Task<AuthenticateResult> AuthenticateAsync(this HttpContext context); // +1 sobrecarga
public static Task ChallengeAsync(this HttpContext context); // +3 sobrecargas
public static Task ForbidAsync(this HttpContext context); // +3 sobrecargas
public static Task SignInAsync(this HttpContext context, ClaimsPrincipal principal); // +3 sobrecargas
public static Task SignOutAsync(this HttpContext context); // +3 sobrecargas
```
También se expone un método con dos sobrecargas para extraer el valor de un token en caso de que exista un token de autenticación en la petición:
```csharp
public static Task<string> GetTokenAsync(this HttpContext context, string scheme, string tokenName);
public static Task<string> GetTokenAsync(this HttpContext context, string tokenName);
```

##### Mediante objetos IActionResult
También podemos usar una serie de objetos para devolver como resultado del método de acción.
Estos objetos se encargarán de invocar la acción correspondiente durante la ejecución de la petición usando el método de extensión sobre HttpContext.
Todos ellos implementan `IActionResult` y cuentan con varias sobrecargas en sus constructores.

```csharp
return new ChallengeResult("oidc");                    // Invocará la acción Challenge para el esquema "oidc"
return new ForbidResult();                             // Invocará la acción Forbid para el esquema configurado por defecto para esta acción
return new SignInResult("Cookies", principal);         // Invocará la acción SignIn para el esquema "Cookies" para persistir principal
return new SignOutResult(new[] { "Cookies", "oidc" }); // Invocará la acción SignOut para los esquemas "Cookies" y "oidc"
```

##### Mediante métodos de ControllerBase
Simplemente son una serie de métodos que crean instancias de los objetos IActionResult del apartado anterior. De esta forma podemos convertir el new `[Action]Result()` en `[Action]()`.
Estos métodos pertenecen a la clase ControllerBase por lo que pueden ser invocados desde el controlador. Al igual que los anteriores también cuentan con varias sobrecargas.
```csharp
return Challenge("oidc");              // Creará un objeto ChallengeResult para el esquema "oidc"
return Forbid("");                     // Creará un objeto ForbidResult para el esquema configurado por defecto para esta acción
return SignIn(principal, "Cookies");   // Creará un objeto SignInResult para el esquema "Cookies" para persistir el principal
return SignOut("Cookies", "oidc");     // Creará un objeto SignOutResult para los esquemas "Cookies" y "oidc"
```

- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- Parte III. ¿Cómo se invocan las acciones?
- **[Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)**
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)