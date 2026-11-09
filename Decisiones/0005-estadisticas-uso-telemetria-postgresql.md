# 0005 - Diseño del Subsistema de Estadísticas de Uso mediante Patrones Observer y Facade

* **Estado:** aceptada
* **Fecha:** 2026-11-09
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
Los administradores y redactores deben consultar cuatro métricas de uso: noticias más leídas, tiempo de permanencia, tasas de rebote y conversiones a suscriptores. La arquitectura vigente sigue el estilo Cliente-Servidor REST (ADR-0001.3), incorpora un `ApiGateway` que distingue PC y Tablet mediante `X-Device-Type` (ADR-0002), mantiene suscripciones anónimas (ADR-0004) y separa los datos en dos bases PostgreSQL 19: una para administración/redacción y otra para suscriptores. Es necesario decidir la arquitectura y los patrones de diseño para capturar y consultar estas métricas sin acoplar la lógica de negocio, sin romper el anonimato y sin añadir infraestructura innecesaria.

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF6 - Estadísticas de uso (Los administradores y redactores pueden consultar las noticias más leídas, el tiempo de permanencia, las tasas de rebote y las conversiones a suscriptores).

## Opciones Consideradas
* **Opción 1:** Patrón `Observer` en memoria local junto con una `Facade` analítica (`ServicioEstadisticas`) sobre la base PostgreSQL 19 de administración y redacción (`AdminRedaccionDAO`).
* **Opción 2:** Patrón `Publish-Subscribe` (Pub-Sub) mediante broker de mensajería en red y almacén analítico separado.
* **Opción 3:** Delegación en plataforma de analítica externa integrada mediante el patrón `Adapter`.

## Decisión Adoptada
Opción elegida: "**Opción 1: Patrones `Observer` y `Facade` sobre PostgreSQL 19 (`AdminRedaccionDAO`)**", porque:
* **Desacopla el dominio mediante `Observer`:** `ServicioNoticias` (eventos de lectura, permanencia y rebote) y `ServicioSuscripciones` (altas confirmadas) actúan como sujetos observados (*Subjects*). Al cambiar su estado, notifican en memoria local a `ServicioEstadisticas` (`Observer`) sin depender de cómo se calculan o almacenan las métricas.
* **Simplifica la consulta mediante `Facade`:** `ServicioEstadisticas` expone una interfaz única y de alto nivel (`Facade`) hacia `EstadisticasController` para consultar las cuatro métricas agregadas, ocultando la lógica interna de agregación.
* **Evita infraestructura distribuida:** Como el backend reside en un único servidor (`BackendServidor`), la comunicación en memoria de `Observer` resuelve RF6 sin introducir un intermediario de red (*broker*).
* **Preserva la separación de datos y el anonimato:** Toda la analítica se guarda únicamente en `AdminRedaccionDAO` segmentada por `X-Device-Type`, sin acceder a `SuscriptoresDAO` ni almacenar identificadores de suscriptor (`tokenHash`).

### Consecuencias
* **Positivas:** Bajo acoplamiento entre negocio y analítica; reutiliza el servidor y las dos bases PostgreSQL 19 existentes; control total de lecturas y conversiones reales sin depender de terceros.
* **Negativas:** Las notificaciones en memoria comparten proceso y carga de escritura con la base PostgreSQL 19 de administración y redacción; si el sistema escalara a múltiples servicios independientes, habría que sustituir `Observer` por `Publish-Subscribe`.
* **Neutras:** Los ASJ deben modelar en `Arquitecturas/` la interfaz observadora, el rol de fachada en `ServicioEstadisticas` y los eventos anónimos de uso en `PkgDatos`.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Patrones `Observer` y `Facade` sobre PostgreSQL 19 (`AdminRedaccionDAO`)
* **A favor (+):** Desacopla `ServicioNoticias` y `ServicioSuscripciones` del subsistema analítico sin añadir un broker externo ni una tercera base de datos.
* **A favor (+):** Centraliza en una fachada (`ServicioEstadisticas`) las cuatro métricas requeridas por RF6 y mantiene el aislamiento total frente a `SuscriptoresDAO`.
* **En contra (-):** Comparte el almacén de administración/redacción en picos de lectura y limita el alcance de la notificación al proceso local del servidor.

### Opción 2: Patrón `Publish-Subscribe` con broker de mensajería en red
* **A favor (+):** Logra desacoplamiento espacial y temporal total entre emisores y consumidores, absorbiendo grandes picos de eventos.
* **En contra (-):** Introduce un middleware de mensajería (MOM) y complejidad distribuida injustificada para un único servidor de aplicaciones.
* **En contra (-):** Requiere infraestructura adicional fuera de las dos bases PostgreSQL 19 definidas en el caso de estudio.

### Opción 3: Analítica externa integrada mediante patrón `Adapter`
* **A favor (+):** Evita diseñar y mantener dentro del backend propio la lógica de cálculo de permanencia, rebote y agregación de métricas.
* **En contra (-):** Expone datos de navegación a un proveedor externo y pierde precisión por bloqueadores de rastreo en el cliente.
* **En contra (-):** Dificulta correlacionar las conversiones reales confirmadas en el backend con la lectura previa de noticias.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar esta decisión en `Arquitecturas/`:
  1. Definir la interfaz `ObservadorEstadisticas` (`<<interface>>`) implementada por `ServicioEstadisticas`, y las relaciones de notificación desde `ServicioNoticias` y `ServicioSuscripciones` (`Subjects` → `Observer`).
  2. Modelar `ServicioEstadisticas` como `Facade` entre `EstadisticasController`/`EstadisticasRestController` (`registrarEventoUso()`, `consultarEstadisticas()`) y la persistencia en `AdminRedaccionDAO`.
  3. Añadir la entidad anónima `EventoUso` en `AdminRedaccionDAO` (tipo de métrica, dispositivo `X-Device-Type`, duración), sin ninguna relación con `SuscriptoresDAO` ni `Suscriptor`.
* **Notas de Discusión ASS-ASC:**
  * *Aceptada tras la revisión de los arquitectos cognitivos.*
 * Durante la revisión, Sergio González (ASC) preguntó si era necesario usar Publish-Subscribe o si Observer era suficiente. También señaló que Observer limita las notificaciones al backend. Sara Guillén (ASC) planteó si guardar los eventos podía generar demasiada carga en PostgreSQL 19 y cómo contar las conversiones sin identificar a los suscriptores.
  * Antonio Machuca y Enrique Aldama (ASS) explicaron que ahora no se necesita añadir un intermediario. Las conversiones se pueden registrar como eventos anónimos sin guardar el token del suscriptor.
  * Después de comentarlo, el equipo decidió mantener Observer y Facade. Se acepta la posible carga en la base de datos como una limitación de la propuesta. Si más adelante fuera necesario distribuir el procesamiento se podría valorar Publish-Subscribe.

## Referencias
* Erich Gamma et al., *Design Patterns: Elements of Reusable Object-Oriented Software* (Patrones `Observer` y `Facade`).
* Refactoring Guru, "Observer Pattern", [https://refactoring.guru/design-patterns/observer](https://refactoring.guru/design-patterns/observer)
* Refactoring Guru, "Facade Pattern", [https://refactoring.guru/design-patterns/facade](https://refactoring.guru/design-patterns/facade)
