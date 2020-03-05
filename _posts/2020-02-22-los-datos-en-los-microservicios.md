---
layout: post
title: Consideraciones sobre los datos en microservicios
---
# Introducción
Una de las ventajas más importantes de una aquitectura basada en microservicios es la independencia y desacoplamiento que existe entre ellos. Nos ofrecen agilidad a la hora de desarrollar y distribuir nuestro software ya que está dividido en funcionalidades pequeñas y aisladas. Las aplicaciones monolíticas nos permiten compartir un único esquema de datos, sin embargo, si nuestro objetivo es el de aislar nuestros microservicios tendremos que cambiar de estrategia en el almacenamiento de datos.

Para mantener un aislamiento completo entre microservicios también deben separase las fuentes de datos para limitar el alcance de futuros cambios en sus esquemas. Por esta razón, uno de los principios básicos en una arquitectura de microservicios dice que cada servicio es el dueño de sus propios datos, es decir, los servicios no comparten sus fuentes de datos. Una ventaja de esta separación es que nos permite utilizar distintas tecnologías de persistencia en cada uno de nuestros servicios según las necesidades individuales (SQL, NoSQL, híbridas...).

Este enfoque nos impone desafíos que no existen en las aplicaciones monolíticas. Algunos ejemplos son: ¿Cómo accedemos a los datos privados de otro servicio? ¿Cómo realizamos consultas que abarquen datos de varios servicios? ¿Cómo realizamos transacciones que impliquen a distintas bases de datos de distintos servicios? ¿Cómo garantizamos la atomicidad entre una actualización de un dato y la publicación de su evento?

## Redundancia de datos ??
Distintos servicios necesitan trabajar con la misma información (o subconjunto de ella), por lo que si replicamos esta información en las bases de datos de nuestros microservicios nos evitamos algunos de los problemas expuestos. Sin embargo nos encontraremos con otros nuevos.

El problema de la redundancia de datos es que puede introducir un nivel de inconsistencia en la información. En una aplicación tradicional un dato sólo está en un sitio, por lo que si cambia, sólo ha de hacerlo en ese sitio. Sin embargo, en el caso de los microservicios tendremos situaciones en las que un mismo dato deberá estar disponible en distintos servicios, por lo que deberemos contar con mecanismos de propagación en nuestras actualizaciones de datos. Si queremos que estos mecanismos no introduzcan esperas que afecten a la **disponibilidad**, debido a las restricciones impuestas en las aplicaciones distribuidas (teorema de CAP), no podremos garantizar la consistencia fuerte. Por esta razón deberemos adoptar la **consistencia eventual** como norma, y, en las ocasiones que necesitemos una consistencia fuerte, deberemos optar por una solución que mantenga la consistencia sacrificando la disponibilidad.

#### Sincronización de datos
Una arquitectura oridentada a eventos podría encargarse de la sincronización de la información. Por ejemplo, la fuente única de verdad se encargaría de realizar las actualizaciones necesarias en la entidad y de enviar un evento de actualización. Los servicios interesados podrían suscribirse a estos eventos y mantener su propia vista materializada. El diseñador debe ser consciente de que estos datos tendrían una cosistencia eventual.

Hay que tener en cuenta que una arquitectura orientada a eventos dificulta la consulta de datos... *****

## Fuente única de la verdad
Podemos exponer la información que necesitamos que sea pública en un microservicio con una API. Aquellos servicios que requieran de información actualizada deberán consumir este servicio. Si otros servicios se conforman con una consistencia eventual podrán mantener su propia fuente de datos. Esta fuente de datos estará sincronizada con la única fuente de verdad de algún modo.

## Event sourcing
Se trata de una forma alternativa de almacenamiento. En lugar de almacenar la información actual de una entidad lo que se guarda es una secuencia de eventos ocurridos sobre la entidad. Los eventos se almacenan en un **Event Store** de forma indefinida. Un servicio puede enviar eventos de actualización de la entidad mientras otros pueden suscribirse y recibir estos eventos para mantener su propia vista materializada. En cualquier momento se puede regenerar el estado actual reproduciendo todos sus eventos.

El hecho de almacenar los eventos de los cambios producidos sobre una entidad ofrece algunas ventajas:
- Se puden realizar de forma muy sencilla rollbacks sobre los cambios producidos. Basta con eliminar eventos del final para ir dejando la entidad en sus estados anteriores. Puede llegar a ser muy útil como **acciones de compensación** que veremos más adelante.
- Tenemos un histórico guardado con todos los cambios producidos. Puede ser una herramienta bastante interesante para ciertos estudios (Big Data, auditorias...).

Este patrón se utiliza a menudo junto con el patrón CQRS ya que le permite sincronizar sus vistas materializadas usadas con fines de lectura.

Mejora la escalabilidad, los servicios que emiten los eventos se mantienen desacoplados de aquellos que los consumen.

*Casos de uso

Aunque nos ofrece atomicidad en las actualizaciones de datos, de nuevo tendremos que manejar consistencia eventual y careceremos de transacciones.

## Consulta de datos
Consiste en tener un servicio consumidor de eventos con una base de datos diseñada sólo para consultas que sea capaz de aportar la información que necesitemos. El agregado de los datos se realiza mediante eventos producidos por las distintas fuentes. Nuestro servicio está suscrito a todos ellos y se encarga de mantener una vista completa de toda la información. En este caso también hay que tener en cuenta que tendremos consistencia eventual.

### Agregado de datos mediante Api Gateway
Cuando es necesario agrupar informaciónd de datos procedentes de varios microservicios podemos optar a realizar este agrupamiento dentro de una misma aplicación. En lugar de realizar el join clásico en base de datos, habrá una aplicación encargada de invocar a los servicios dueños de la información y de realizar esta unión. Normalmente los Api Gateway son los encargados de realizar esta agregación de datos.

El problema puede darse cuando falla uno de los servicios

### Vistas CQRS


## Transacciones
Utilizar transacciones distribuidas entre bases de datos que pertenecen a distintos servicios no suele ser una opción. Este tipo de transacciones suelen impactar bastante en el rendimiento y la concurrencia del sistema ya que se producirían bloqueos. Por eso, si necesitamos realizar acciones en varias bases de datos de forma transaccional deberemos utilizar un patrón de transacciones de larga vida (long-lived transactions).

Uno de estos patrones es el patrón **saga**, que divide una transacción distribuida en otras transacciones más pequeñas de forma que puedan ser controladas por el mismo microservicio de forma local y por lo tanto, sí puedan ser transaccionales. De esta manera se podrán ejecutar sin bloqueos largos en el tiempo. El diseño debe contemplar la posibilidad de rollback en cualquiera de las transacciones, por lo que deberemos implementar **acciones de compensación** para deshacer los pasos dados hasta que se inició el rollback.

Una vez más, una aquitectura orientada a eventos nos facilitaría la coordinación del proceso completo. Existen dos tipos de coordinación para sagas: **coreografía** y **orquestación**.


**Separar cuando contamos de una arquitectura orientada a eventos de cuando no la tenemos.
