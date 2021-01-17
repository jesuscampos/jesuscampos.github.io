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
- Parte IV. ¿Quién invoca las acciones?
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)

## ¿Quién invoca las acciones?
Como ya he comentado, el propio framework nos ahorrará el tener que invocar muchas de estas acciones.
Solo en algunos momentos nos interesará invocar estas acciones desde nuestro código. Por ejemplo:

- Tras un clic en un botón "Login" para un esquema remoto podemos llamar a la acción **Challenge** para que inicie su proceso de autenticación remota.
  De esta forma se producirá una redirección a una página de login externa configurada en el handler mediante sus opciones.
  
  En el caso de un login local resulta más sencillo ir a la página de login local directamente sin la necesidad de invocar esta acción.
  
  >De todas formas, el framework de autorización se encargará de invocar esta acción por nosotros en caso de que el usuario intente acceder a un recurso protegido sin estar autenticado.
  
```csharp
// Enlace a login local
<a asp-controller="Home" asp-action="Login">Login</a>
```
```csharp
// Invocación de la acción challenge para un handler remoto. El handler nos redireccionará a la página de login remoto
public ActionResult Login(string returnUrl)
{
    return Challenge("oidc");  // Si tenemos configurado por defecto el esquema "oidc" para la acción Challenge no sería necesario el parámetro
}
``` 

- Tras un post de un login local validado con éxito y creada una identidad `ClaimsPrincipal` deberemos llamar a la acción **SignIn** para que cree una sesión del usuario para mantenerla activa entre llamadas.
  
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
Podemos ver que la acción en **Authenticate** (Creación del ClaimsPrincipal) es completamente manual ya que somos nosotros quienes acabamos de validar y de obtener la información del usuario.

Tal y como veremos, en el caso de un login remoto será el propio handler remoto quien realice una **"autenticación remota"** tras la captura de la petición cuando regrese tras una autenticación en el proveedor de identidad externo.

- Tras un clic en un botón de "Logout" deberemos llamar a la acción **SignOut** para que elimine la persistencia (sesión de usuario), por ejemplo, que se elimine la Cookie de sesión. En el caso de tener un handler remoto el **SignOut** lo deberemos hacer tanto para el esquema local (para que elimine la sesión local) como para el esquema remoto, ya que esto iniciará el proceso de SignOut del handler remoto, que no es más que una redirección a la página de "Logout" del servicio Autoridad con el que se realizó el **SingIn**.
```csharp
// SignOut local. Como no indicamos esquema usará el esquema por defecto para la acción SignIn
public async Task<IActionResult> Logout()
{
    await HttpContext.SignOutAsync();
    return RedirectToAction("Login");
}
```
```csharp
// SignOut local y remoto. Debemos avisar también al handler remoto para que inicie su proceso de **SignOut**
public async Task<IActionResult> Logout()
{
    return SignOut(new[] { "Cookies", "oidc" });
}
```

El framework de autorización también puede invocar ciertas acciones de forma transparente para nosotros cuando así lo estime oportuno. Ejemplos:
- Invocar la acción **Challenge** si no hay usuario autenticado y el endpoint al que se accede requiere autenticación de usuario.
- Invocar la acción **Forbid** si hay usuario autenticado pero no cumple con las políticas de autorización para el endpoint al que se accede.

Por último, para ver como son invocadas las distintas acciones "ocultas" que realiza el framework de autenticación veremos más profundamente el middleware de autenticación.

- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- [Parte II. Configuración de la autenticación](2020-09-07-autenticacion-en-asp-net-core-parte2.md)
- [Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)
- Parte IV. ¿Quién invoca las acciones?
- **[Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)**
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md) 