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
- Parte II. Configuraci�n de la autenticaci�n
- **[Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)**
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)

## Configuraci�n de la autenticaci�n
Para configurar la autenticaci�n de nuestras aplicaciones usaremos el m�todo de extensi�n `IServiceCollection.AddAuthentication()`.
Este m�todo, adem�s de registrar ciertas dependencias relacionadas con la autenticaci�n, nos permitir� configurar los esquemas para cada acci�n y asociar los handlers a estos esquemas.

#### Vinculaci�n de esquemas con acciones:

Cuando la aplicaci�n requiere invocar una de estas acciones lo deber� hacer seg�n un esquema concreto, el cual determinar� la forma en que se realice esta acci�n.
Este esquema se puede especificar en la misma invocaci�n a la acci�n como un par�metro o bien, si no se especifica ning�n par�metro, se utilizar� el **esquema por defecto** para esa acci�n.
En el apartado [�Y qui�n invoca las acciones?](#�Y-quin-invoca-las-acciones?) se muestran diferentes sobrecargas que permiten que este par�metro sea opcional.

Estos esquemas los especificamos en la llamada a `AddAuthentication` donde le pasaremos un objeto `Action<AuthenticationOptions>` el cual cuenta con propiedades para todos los tipos de acciones.
Adem�s, cuenta con la propiedad `DefaultScheme` donde le diremos el esquema a utilizar para todas las acciones que no se configuren de forma expl�cita:

```csharp
DefaultScheme
DefaultAuthenticateScheme
DefaultChallengeScheme
DefaultForbidScheme
DefaultSignInScheme
DefaultSignOutScheme
```

A continuaci�n vemos un ejemplo de configuraci�n de los esquemas por defecto para una aplicaci�n web MVC:

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies"; 
    options.DefaultChallengeScheme = "oidc";
})
```

En esta configuraci�n hemos asignado `"Cookies"` como `DafaultScheme`, es decir, este ser� el esquema que se utilizar� para las acciones en las que no especifiquemos un esquema.

Seguidamente asignamos el esquema `"oidc"` como esquema por defecto para la acci�n **Challenge**. Es decir, cualquier invocaci�n a la acci�n **Challenge** usar� el esquema `"oidc"`, y el resto de acciones ser�n gestionadas por el esquema `"Cookies"`.

>Aunque solo podemos configurar un esquema por defecto para cada acci�n, podremos configurar todos los esquemas que consideremos necesarios en nuestra aplicaci�n. Eso s�, a la hora de invocar las acciones de estos esquemas habr� que hacerlo especificando su nombre de esquema.

#### Asignaci�n de handlers a esquema
El m�todo `AddAuthentication` devuelve un objeto `AuthenticationBuilder` con el cual podremos encadenar una serie de m�todos de extensi�n para registrar los handlers de autenticaci�n de nuestra aplicaci�n (`AddCookies`, `AddGoogle`, `AddJwtBearer`...). Y es que realmente hasta ahora solo hemos mapeado unos `string` a los esquemas por defecto de cada acci�n. Todav�a queda especificar lo m�s importante:
**�qu� c�digo debe ejecutarse para cada esquema?**, o lo que es lo mismo, **�c�mo relaciono cada handler de autenticaci�n con estos esquemas?**

Veamos varios ejemplos de asignac��n de handlers a los esquemas correspondientes para distintos escenarios.

##### Cookies y oidc

Continuando con el ejemplo anterior donde ya hab�amos configurado los esquemas por defecto, a�adiremos los m�todos de extensi�n donde relacionaremos cada esquema con su handler.

```csharp
// Combinamos Cookies + oidc
services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies"; 
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", options =>
{
    options.SignInScheme = "Cookies";
    options.Authority = "https://autoridad.com";
    options.ClientId = "clientmvc";
    options.ClientSecret = "secret";
    options.SaveTokens = true;
});
```
`AddCookie` configura un handler de tipo `CookieAuthenticationHandler` para el esquema "Cookies", y `AddOpenIdConnect` configura un handler de tipo `OpenIdConnectHandler` para el esquema "oidc".
Ambos m�todos aceptan un objeto de opciones como par�metro para poder configurar cada handler: `CookieAuthenticationOptions` y `OpenIdConnectOptions` respectivamente.
Para el esquema de "Cookies" no se han pasado opciones, por lo que se usar�n las opciones por defecto.

El primer handler utiliza un modelo de autenticaci�n basado en Cookies de sesi�n para cualquier acci�n de autenticaci�n (excepto Challenge).
El segundo se trata de un handler remoto OpendId para la acci�n Challenge.
Es una configuraci�n muy habitual en aplicaciones web donde contamos con un servicio proveedor de identidad externo, donde se delega la autenticaci�n de usuarios.
M�s adelante describiremos como es todo el flujo de autenticaci�n usando uno de estos servicios remotos, donde el usuario es redireccionado a un servidor donde se identificar� para posteriormente, ser redireccionado de vuelta a nuestra aplicaci�n.

##### Cookies y oauth

Si useamos OAuth2 en vez de OpenId Connect la configuraci�n es bastante similar.
Para el ejemplo anterior, en lugar de utilizar una �nica url para la autoridad deberemos especificar las urls de los endpoints **token** y **authorization**:

```csharp
    options.AuthorizationEndpoint = "https://autoridad.com/account/authorize";
    options.TokenEndpoint = "https://autoridad.com/account/token";
```

##### Cookies

Se trata de una simplificaci�n de las anteriores ya que todas las acciones las manejar� el handler de `"Cookies"`. Al no tener un handler remoto ser� la propia aplicaci�n quien gestione la autenticaci�n de forma local.
Una acci�n **Challenge** con un esquema basado en Cookies nos redireccionar� a una p�gina local de login.
En el siguiente ejemplo hemos usado las opciones del handler para establecer el path de la vista de login que por defecto apuntaba a "/Account/Login".

```csharp
services.AddAuthentication("Cookies")
.AddCookie(options => 
    {
        options.LoginPath = "Users/Login";
    });
```

Como novedad vemos que existe una sobrecarga de `AddAuthentication` donde pasamos simplemente un string con el nombre del esquema. Este ser� asignado al `DefautlScheme`.

##### Bearer con token jwt

El siguiente ejemplo lo podemos utilizar en una API securizada con tokens jwt.

```csharp
// Bearer
services.AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
       options.Authority = "https://autoridad.com";
       options.Audience = "api1";
    });
```

El m�todo de extensi�n `AddJwtBearer` a�ade un handler de tipo `JwtBearerHandler`, el cual est� dise�ado para validar un token de tipo jwt que deber� ser proporcionado mediante la cabecera **Authorization**.
En este caso no es necesario especificar el nombre del esquema "Bearer" al a�adir el handler debido a que estamos configurando un �nico esquema, por lo que esta relaci�n la har� autom�ticamente.
En cambio, s� que le pasamos un objeto de opciones `JwtBearerOptions` donde especificamos la configuraci�n que necesita el handler.

##### Bearer con token de referencia

Los tokens de referencia son tokens �pacos que no contienen ning�n tipo de informaci�n. Simplemente son una referencia a un token real donde s� contiene toda la informaci�n.
Este token real queda almacenado en el proveedor de identidad. S�lo la API para la que se emiti� el token podr� requerir esta informaci�n al proveedor de identidad mediante una petici�n a la que llamamos [**introspecci�n**](https://tools.ietf.org/html/rfc7662).

Aunque se ha extendido el uso del paquete [IdentityServer4.AccessTokenValidation](https://www.nuget.org/packages/IdentityServer4.AccessTokenValidation/) donde se ofrec�a un �nico handler capaz de tratar tanto tokens de tipo jwt como de referencia finalmente ha quedado obsoleto.

Para tratar con tokens de referencia deberemos utilizar el paquete [IdentityModel.AspNetCore.OAuth2Introspection](https://www.nuget.org/packages/IdentityModel.AspNetCore.OAuth2Introspection/).

```csharp
services.AddAuthentication("Bearer")
    .AddOAuth2Introspection("Bearer", options =>
    {
        options.Authority = "https://autoridad.com";
        options.ClientId = "api1";
        options.ClientSecret = "secret";
    });
```

##### Bearer con tokens jwt y referencia

Para el caso en el que tengamos la posibilidad de recibir ambos tipos de tokens necesitaremos configurar los dos handlers.
Estableceremos como esquema por defecto `"token"` pero configuraremos un **selector** en la propiedad `ForwardDefaultSelector`.
En esta propiedad le pasaremos una funci�n que inpeccionar� la petici�n actual y devolver� el esquema a la que debe delegar la acci�n.
Para nuestro caso el selector deber� distinguir entre un token tipo jwt y uno de referencia.

En el paquete [IdentityModel.AspNetCore.AccessTokenValidation](https://github.com/IdentityModel/IdentityModel.AspNetCore.AccessTokenValidation/blob/main/src/Selector.cs) tenemos un ejemplo de selector que basa su decisi�n en si el token contiene o no el caracter '.', separador utilizado en los tokens jwt.

```csharp
services.AddAuthentication("token")

    // JWT tokens
    .AddJwtBearer("token", options =>
    {
        options.Authority = "https://autoridad.com";
        options.Audience = "api1";

        options.ForwardDefaultSelector = Selector.ForwardReferenceToken("introspection");
    })

    // reference tokens
    .AddOAuth2Introspection("introspection", options =>
    {
        options.Authority = "https://autoridad.com";

        options.ClientId = "api1";
        options.ClientSecret = "secret";
    });
```

### Configuraci�n del middleware
Una vez configurados los servicios, esquemas, handlers y sus opciones, tendremos que asegurarnos que en cada petici�n se trate de autenticar al usuario.
Para ello deberemos insertar el middleware de autenticaci�n `AuthenticationMiddleware` en nuestro pipeline.
Esto lo deberemos hacer antes de que la informaci�n del usuario sea necesaria, es decir, antes de que la autorizaci�n haga sus validaciones y antes de que se pase la petici�n a los controladores.

```csharp
// NetCore 2.1 
app.UseAuthentication();
app.UseMvc();
```
```csharp
// NetCore >= 3.0
app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();

app.UseEndpoints(endpoints => {
   endpoints.MapControllers();
});
```

El middleware, antes de intentar autenticar al usuario de forma local, dar� la oportunidad a los posibles **handlers remotos** de capturar la petici�n por si se trata de una petici�n para ellos.
M�s adelante cuando veamos los flujos para un handler remoto veremos que tipo de peticiones son *capturables* por el handler remoto.

Si no hay handlers remotos o si la petici�n no ha sido capturada por ellos, continuar� intentando la autenticaci�n local con el esquema configurado por defecto para la acci�n **Authenicate**.
Si esta acci�n consigue crear un *ticket* se asignar� su `ClaimsPrincipal` a la propiedad `HttpContext.User` para que est� disponible para el resto del pipeline.

El handler asignado para el esquema utilizado se encargar� de extraer la informaci�n de usuario de la fuente origen.
Si por ejemplo se trata de una handler `CookieAuthenticationHandler` lo har� de la **cookie de sesi�n** entrante, y si el handler es `JwtBearerHandler` lo har� de la cabecera **Authorization**.

Si no hay �xito en la autenticaci�n la propiedad `HttpContext.User` quedar� nula y la petici�n se tratar� como an�nima. S�lo ser�n accesibles los recursos sin securizar.

- [Parte I. Introducci�n](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- Parte II. Configuraci�n de la autenticaci�n
- **[Parte III. �C�mo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)**
- [Parte IV. �Qui�n invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticaci�n](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)