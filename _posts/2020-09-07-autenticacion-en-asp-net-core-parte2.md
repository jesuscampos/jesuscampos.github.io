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
- Parte II. Configuración de la autenticación
- **[Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)**
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)

## Configuración de la autenticación
Para configurar la autenticación de nuestras aplicaciones usaremos el método de extensión `IServiceCollection.AddAuthentication()`.
Este método, además de registrar ciertas dependencias relacionadas con la autenticación, nos permitirá configurar los esquemas para cada acción y asociar los handlers a estos esquemas.

#### Vinculación de esquemas con acciones:

Cuando la aplicación requiere invocar una de estas acciones lo deberá hacer según un esquema concreto, el cual determinará la forma en que se realice esta acción.
Este esquema se puede especificar en la misma invocación a la acción como un parámetro o bien, si no se especifica ningún parámetro, se utilizará el **esquema por defecto** para esa acción.
En el apartado [¿Y quién invoca las acciones?](#¿Y-quin-invoca-las-acciones?) se muestran diferentes sobrecargas que permiten que este parámetro sea opcional.

Estos esquemas los especificamos en la llamada a `AddAuthentication` donde le pasaremos un objeto `Action<AuthenticationOptions>` el cual cuenta con propiedades para todos los tipos de acciones.
Además, cuenta con la propiedad `DefaultScheme` donde le diremos el esquema a utilizar para todas las acciones que no se configuren de forma explícita:

```csharp
DefaultScheme
DefaultAuthenticateScheme
DefaultChallengeScheme
DefaultForbidScheme
DefaultSignInScheme
DefaultSignOutScheme
```

A continuación vemos un ejemplo de configuración de los esquemas por defecto para una aplicación web MVC:

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies"; 
    options.DefaultChallengeScheme = "oidc";
})
```

En esta configuración hemos asignado `"Cookies"` como `DafaultScheme`, es decir, este será el esquema que se utilizará para las acciones en las que no especifiquemos un esquema.

Seguidamente asignamos el esquema `"oidc"` como esquema por defecto para la acción **Challenge**. Es decir, cualquier invocación a la acción **Challenge** usará el esquema `"oidc"`, y el resto de acciones serán gestionadas por el esquema `"Cookies"`.

>Aunque solo podemos configurar un esquema por defecto para cada acción, podremos configurar todos los esquemas que consideremos necesarios en nuestra aplicación. Eso sí, a la hora de invocar las acciones de estos esquemas habrá que hacerlo especificando su nombre de esquema.

#### Asignación de handlers a esquema
El método `AddAuthentication` devuelve un objeto `AuthenticationBuilder` con el cual podremos encadenar una serie de métodos de extensión para registrar los handlers de autenticación de nuestra aplicación (`AddCookies`, `AddGoogle`, `AddJwtBearer`...). Y es que realmente hasta ahora solo hemos mapeado unos `string` a los esquemas por defecto de cada acción. Todavía queda especificar lo más importante:
**¿qué código debe ejecutarse para cada esquema?**, o lo que es lo mismo, **¿cómo relaciono cada handler de autenticación con estos esquemas?**

Veamos varios ejemplos de asignacíón de handlers a los esquemas correspondientes para distintos escenarios.

##### Cookies y oidc

Continuando con el ejemplo anterior donde ya habíamos configurado los esquemas por defecto, añadiremos los métodos de extensión donde relacionaremos cada esquema con su handler.

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
Ambos métodos aceptan un objeto de opciones como parámetro para poder configurar cada handler: `CookieAuthenticationOptions` y `OpenIdConnectOptions` respectivamente.
Para el esquema de "Cookies" no se han pasado opciones, por lo que se usarán las opciones por defecto.

El primer handler utiliza un modelo de autenticación basado en Cookies de sesión para cualquier acción de autenticación (excepto Challenge).
El segundo se trata de un handler remoto OpendId para la acción Challenge.
Es una configuración muy habitual en aplicaciones web donde contamos con un servicio proveedor de identidad externo, donde se delega la autenticación de usuarios.
Más adelante describiremos como es todo el flujo de autenticación usando uno de estos servicios remotos, donde el usuario es redireccionado a un servidor donde se identificará para posteriormente, ser redireccionado de vuelta a nuestra aplicación.

##### Cookies y oauth

Si useamos OAuth2 en vez de OpenId Connect la configuración es bastante similar.
Para el ejemplo anterior, en lugar de utilizar una única url para la autoridad deberemos especificar las urls de los endpoints **token** y **authorization**:

```csharp
    options.AuthorizationEndpoint = "https://autoridad.com/account/authorize";
    options.TokenEndpoint = "https://autoridad.com/account/token";
```

##### Cookies

Se trata de una simplificación de las anteriores ya que todas las acciones las manejará el handler de `"Cookies"`. Al no tener un handler remoto será la propia aplicación quien gestione la autenticación de forma local.
Una acción **Challenge** con un esquema basado en Cookies nos redireccionará a una página local de login.
En el siguiente ejemplo hemos usado las opciones del handler para establecer el path de la vista de login que por defecto apuntaba a "/Account/Login".

```csharp
services.AddAuthentication("Cookies")
.AddCookie(options => 
    {
        options.LoginPath = "Users/Login";
    });
```

Como novedad vemos que existe una sobrecarga de `AddAuthentication` donde pasamos simplemente un string con el nombre del esquema. Este será asignado al `DefautlScheme`.

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

El método de extensión `AddJwtBearer` añade un handler de tipo `JwtBearerHandler`, el cual está diseñado para validar un token de tipo jwt que deberá ser proporcionado mediante la cabecera **Authorization**.
En este caso no es necesario especificar el nombre del esquema "Bearer" al añadir el handler debido a que estamos configurando un único esquema, por lo que esta relación la hará automáticamente.
En cambio, sí que le pasamos un objeto de opciones `JwtBearerOptions` donde especificamos la configuración que necesita el handler.

##### Bearer con token de referencia

Los tokens de referencia son tokens ópacos que no contienen ningún tipo de información. Simplemente son una referencia a un token real donde sí contiene toda la información.
Este token real queda almacenado en el proveedor de identidad. Sólo la API para la que se emitió el token podrá requerir esta información al proveedor de identidad mediante una petición a la que llamamos [**introspección**](https://tools.ietf.org/html/rfc7662).

Aunque se ha extendido el uso del paquete [IdentityServer4.AccessTokenValidation](https://www.nuget.org/packages/IdentityServer4.AccessTokenValidation/) donde se ofrecía un único handler capaz de tratar tanto tokens de tipo jwt como de referencia finalmente ha quedado obsoleto.

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
En esta propiedad le pasaremos una función que inpeccionará la petición actual y devolverá el esquema a la que debe delegar la acción.
Para nuestro caso el selector deberá distinguir entre un token tipo jwt y uno de referencia.

En el paquete [IdentityModel.AspNetCore.AccessTokenValidation](https://github.com/IdentityModel/IdentityModel.AspNetCore.AccessTokenValidation/blob/main/src/Selector.cs) tenemos un ejemplo de selector que basa su decisión en si el token contiene o no el caracter '.', separador utilizado en los tokens jwt.

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

### Configuración del middleware
Una vez configurados los servicios, esquemas, handlers y sus opciones, tendremos que asegurarnos que en cada petición se trate de autenticar al usuario.
Para ello deberemos insertar el middleware de autenticación `AuthenticationMiddleware` en nuestro pipeline.
Esto lo deberemos hacer antes de que la información del usuario sea necesaria, es decir, antes de que la autorización haga sus validaciones y antes de que se pase la petición a los controladores.

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

El middleware, antes de intentar autenticar al usuario de forma local, dará la oportunidad a los posibles **handlers remotos** de capturar la petición por si se trata de una petición para ellos.
Más adelante cuando veamos los flujos para un handler remoto veremos que tipo de peticiones son *capturables* por el handler remoto.

Si no hay handlers remotos o si la petición no ha sido capturada por ellos, continuará intentando la autenticación local con el esquema configurado por defecto para la acción **Authenicate**.
Si esta acción consigue crear un *ticket* se asignará su `ClaimsPrincipal` a la propiedad `HttpContext.User` para que esté disponible para el resto del pipeline.

El handler asignado para el esquema utilizado se encargará de extraer la información de usuario de la fuente origen.
Si por ejemplo se trata de una handler `CookieAuthenticationHandler` lo hará de la **cookie de sesión** entrante, y si el handler es `JwtBearerHandler` lo hará de la cabecera **Authorization**.

Si no hay éxito en la autenticación la propiedad `HttpContext.User` quedará nula y la petición se tratará como anónima. Sólo serán accesibles los recursos sin securizar.

- [Parte I. Introducción](2020-09-07-autenticacion-en-asp-net-core-parte1.md)
- Parte II. Configuración de la autenticación
- **[Parte III. ¿Cómo se invocan las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte3.md)**
- [Parte IV. ¿Quién invoca las acciones?](2020-09-07-autenticacion-en-asp-net-core-parte4.md)
- [Parte V. Middleware de autenticación](2020-09-07-autenticacion-en-asp-net-core-parte5.md)
- [Parte VI. Nota sobre redirect uri's](2020-09-07-autenticacion-en-asp-net-core-parte6.md)