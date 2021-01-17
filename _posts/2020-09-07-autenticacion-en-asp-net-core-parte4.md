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
- Parte IV. �Qui�n invoca las acciones?
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)

## �Qui�n invoca las acciones?
Como ya he comentado, el propio framework nos ahorrar� el tener que invocar muchas de estas acciones.
Solo en algunos momentos nos interesar� invocar estas acciones desde nuestro c�digo. Por ejemplo:

- Tras un clic en un bot�n "Login" para un esquema remoto podemos llamar a la acci�n **Challenge** para que inicie su proceso de autenticaci�n remota.
  De esta forma se producir� una redirecci�n a una p�gina de login externa configurada en el handler mediante sus opciones.
  
  En el caso de un login local resulta m�s sencillo ir a la p�gina de login local directamente sin la necesidad de invocar esta acci�n.
  
  >De todas formas, el framework de autorizaci�n se encargar� de invocar esta acci�n por nosotros en caso de que el usuario intente acceder a un recurso protegido sin estar autenticado.
  
```csharp
// Enlace a login local
<a asp-controller="Home" asp-action="Login">Login</a>
```
```csharp
// Invocaci�n de la acci�n challenge para un handler remoto. El handler nos redireccionar� a la p�gina de login remoto
public ActionResult Login(string returnUrl)
{
    return Challenge("oidc");  // Si tenemos configurado por defecto el esquema "oidc" para la acci�n Challenge no ser�a necesario el par�metro
}
``` 

- Tras un post de un login local validado con �xito y creada una identidad `ClaimsPrincipal` deberemos llamar a la acci�n **SignIn** para que cree una sesi�n del usuario para mantenerla activa entre llamadas.
  
```csharp
[HttpPost]
public async Task<IActionResult> Login(User user)
{
    var user = _users.Value.Find(u => u.UserName == user.UserName && u.Password == user.Password);

    if (!(user is null))
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.Name, user.UserName),
            new Claim(ClaimTypes.Role, "Admin"),
            // ...
        };
        var identity = new ClaimsIdentity(claims, "Cookies");
        var principal = new ClaimsPrincipal(identity);
        await HttpContext.SignInAsync(principal);

        return RedirectToAction("Index", "Home");
    }
}
```
Podemos ver que la acci�n en **Authenticate** (Creaci�n del ClaimsPrincipal) es completamente manual ya que somos nosotros quienes acabamos de validar y de obtener la informaci�n del usuario.

Tal y como veremos, en el caso de un login remoto ser� el propio handler remoto quien realice una **"autenticaci�n remota"** tras la captura de la petici�n cuando regrese tras una autenticaci�n en el proveedor de identidad externo.

- Tras un clic en un bot�n de "Logout" deberemos llamar a la acci�n **SignOut** para que elimine la persistencia (sesi�n de usuario), por ejemplo, que se elimine la Cookie de sesi�n. En el caso de tener un handler remoto el **SignOut** lo deberemos hacer tanto para el esquema local (para que elimine la sesi�n local) como para el esquema remoto, ya que esto iniciar� el proceso de SignOut del handler remoto, que no es m�s que una redirecci�n a la p�gina de "Logout" del servicio Autoridad con el que se realiz� el **SingIn**.
```csharp
// SignOut local. Como no indicamos esquema usar� el esquema por defecto para la acci�n SignIn
public async Task<IActionResult> Logout()
{
    await HttpContext.SignOutAsync();
    return RedirectToAction("Login");
}
```
```csharp
// SignOut local y remoto. Debemos avisar tambi�n al handler remoto para que inicie su proceso de **SignOut**
public async Task<IActionResult> Logout()
{
    return SignOut(new[] { "Cookies", "oidc" });
}
```

El framework de autorizaci�n tambi�n puede invocar ciertas acciones de forma transparente para nosotros cuando as� lo estime oportuno. Ejemplos:
- Invocar la acci�n **Challenge** si no hay usuario autenticado y el endpoint al que se accede requiere autenticaci�n de usuario.
- Invocar la acci�n **Forbid** si hay usuario autenticado pero no cumple con las pol�ticas de autorizaci�n para el endpoint al que se accede.

Por �ltimo, para ver como son invocadas las distintas acciones "ocultas" que realiza el framework de autenticaci�n veremos m�s profundamente el middleware de autenticaci�n.

- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuraci�n de la autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- Parte IV. �Qui�n invoca las acciones?
- **[Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)**
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md) 