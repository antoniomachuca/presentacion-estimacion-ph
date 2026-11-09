# 0004 - Canal de Suscripciones Temáticas y Control de Acceso Anónimo (Publish-Subscribe y Proxy de Protección)

* **Estado:** aceptada
* **Fecha:** 2026-11-09
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
El periódico digital exige que los lectores puedan suscribirse y darse de baja a 5 categorías de noticias (política, economía, sociedad, nacionales e internacionales) de forma completamente anónima (RF2). El sistema tiene prohibido registrar datos personales identificables (nombre, DNI, correo electrónico), pero debe ser capaz de discriminar si una petición tiene derecho a acceder a los artículos de una sección contratada.

Desde el punto de vista del diseño arquitectónico, el problema consiste en:
1. ¿Cómo desacoplar la publicación de noticias en las 5 secciones de la entrega a múltiples lectores concurrentes?
2. ¿Cómo interponer un mecanismo de control de acceso que valide los derechos de suscripción sin introducir un subsistema de autenticación pesado ni almacenar identidades en el servidor?

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF2 - Suscripción y baja anónima (Los lectores pueden suscribirse y darse de baja a diferentes categorías de noticias de forma anónima).
* **Restricción de Diseño Arquitectónico:** Mantener un bajo acoplamiento (*loosely coupling*) y alta escalabilidad horizontal entre la capa editorial y los suscriptores, preservando el modelo sin estado (*stateless*) sobre HTTP/REST (RD2).

## Opciones Consideradas
* **Opción 1:** Patrón Publish-Subscribe distribuido por categorías con Proxy de Protección (*Protection Proxy*) para control de acceso anónimo.
* **Opción 2:** Patrón Mediator centralizado con registro de sesiones activas en el servidor.
* **Opción 3:** Consulta síncrona directa punto a punto (*Point-to-Point Polling*) sin abstracción de canales de suscripción.

## Decisión Adoptada
Elegimos la **Opción 1: Patrón Publish-Subscribe distribuido por categorías con Proxy de Protección (*Protection Proxy*)**, porque:
* Aplica el **Patrón Publish-Subscribe** tratando las 5 secciones de noticias como canales temáticos independientes (*topics*). Los redactores publican en los canales y los lectores anónimos consumen de ellos, logrando un desacoplamiento estructural completo (*loosely coupling*).
* Incorpora un **Patrón Proxy de Protección** como intermediario ante el servicio de noticias. El proxy intercepta las solicitudes de lectura, comprueba la validez del identificador opaco de suscripción contra el repositorio y autoriza el acceso a la categoría correspondiente sin requerir datos personales.
* Facilita el proceso de baja: darse de baja consiste simplemente en revocar la vinculación del identificador anónimo respecto al canal temático en el proxy, sin necesidad de procesos complejos de borrado de cuentas de usuario.

### Consecuencias
* **Positivas:** Máximo desacoplamiento entre el módulo editorial y los suscriptores; el sistema escala de forma natural ante picos de lectura en cualquiera de las 5 secciones; cumplimiento riguroso del anonimato por diseño (el dominio solo maneja tokens de acceso y canales temáticos, cero datos personales).
* **Negativas:** Introduce clases intermedias en el diseño (el interfaz del Proxy y los manejadores de eventos temáticos) que deben ser documentadas en el diagrama de clases UML.
* **Neutras:** Los ASJ deben modelar la relación entre el componente de entrada, el Proxy de Protección y los canales temáticos de noticias.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Publish-Subscribe con Proxy de Protección
* **A favor (+):** Diseñado específicamente para sistemas de suscripción de noticias con alta concurrencia, asegurando escalabilidad y bajo acoplamiento.
* **A favor (+):** El Proxy de Protección aísla el servicio de noticias (`ServicioNoticias`) de la lógica de verificación de derechos, cumpliendo el principio de responsabilidad única.
* **A favor (+):** Preserva el anonimato absoluto: el canal temático solo conoce identificadores de suscripción, nunca personas físicas.
* **En contra (-):** Mayor número de componentes abstractos a modelar en el diagrama de clases en comparación con una llamada directa.

### Opción 2: Patrón Mediator centralizado con sesiones activas
* **A favor (+):** Centraliza la orquestación de mensajes y usuarios en un único componente mediador.
* **En contra (-):** Rompe el principio de diseño *stateless* de la arquitectura REST al forzar el mantenimiento de un registro de sesiones activas en el servidor.
* **En contra (-):** El mediador se convierte en un cuello de botella arquitectónico al concentrar la publicación de noticias, las altas/bajas y la validación de credenciales.

### Opción 3: Consulta síncrona punto a punto sin canales
* **A favor (+):** Diseño trivial sin capas intermedias.
* **En contra (-):** Acopla fuertemente el cliente a las tablas internas de noticias, obligando a realizar consultas constantes para comprobar novedades.
* **En contra (-):** Nula escalabilidad ante miles de lectores concurrentes en noticias de última hora.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar en el diagrama de clases y paquetes de `Arquitecturas/`:
  1. La interfaz `INoticiasService` y su implementación concreta `NoticiasServiceImpl`.
  2. La clase `NoticiasProtectionProxy` intermediando el acceso a `NoticiasServiceImpl` y validando el identificador opaco.
  3. Los canales temáticos de suscripción (`CanalTematico`) representando las 5 secciones (política, economía, sociedad, nacionales e internacionales) bajo la estructura del patrón Publish-Subscribe.
* **Notas de Discusión ASS-ASC:**
  * *Intervención de Sergio González (ASC):* Sergio advirtió que la propuesta inicial del equipo se centraba en detalles de implementación (hashes, almacenamiento local y librerías) en lugar de una solución de diseño arquitectónico. Propuso estructurar el canal de suscripciones temáticas mediante el **Patrón Publish-Subscribe** (siguiendo las directrices del Tema 2 de la asignatura) y resolver el control de acceso anónimo mediante un **Proxy de Protección**.
  * *Intervención de Sara Guillen (ASC):* Sara respaldó la propuesta señalando que el Proxy de Protección descarga al `NoticiasController` de verificar suscripciones y permite que las 5 secciones funcionen como tópicos desacoplados.
  * *Resolución ASS (Antonio Machuca y Enrique Aldama):* Los ASS aceptaron la corrección metodológica de Sergio, descartaron el redactado de implementación y aprobaron esta solución de diseño basada en patrones.

## Referencias
* Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides (GoF), "Design Patterns: Elements of Reusable Object-Oriented Software" (Patrones Proxy y Observer).