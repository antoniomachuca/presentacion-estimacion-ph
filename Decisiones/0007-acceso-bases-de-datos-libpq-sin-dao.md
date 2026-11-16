# 0007 - Acceso a la Base de Datos PostgreSQL 19 mediante el Componente de Acceso de PostgreSQL (sin capa DAO)

* **Estado:** aceptada
* **Fecha:** 2026-11-16
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
Hasta ahora los servicios del backend accedían a la base de datos PostgreSQL 19 a través de una capa DAO (`AdminRedaccionDAO` y `SuscriptoresDAO`). Tras consultarlo con el profesor, esta capa no debe existir: los servicios deben acceder a la base de datos directamente mediante el componente de acceso que describe la documentación de PostgreSQL. La pregunta de diseño es cómo se relacionan los servicios con la base de datos a través de ese componente.

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RD3 - Bases de datos con PostgreSQL 19 (El sistema debe utilizar bases de datos PostgreSQL 19).
* Indicación del profesor: sustituir la capa DAO por el componente de acceso de la documentación de PostgreSQL.
* Restricción del sistema: mantener el anonimato de los suscriptores (ADR-0004, ADR-0005).

## Opciones Consideradas
* **Opción 1:** Los servicios acceden directamente a la base de datos mediante el componente de acceso de PostgreSQL, sin capa intermedia.
* **Opción 2:** Mantener la capa DAO entre los servicios y el componente de acceso.
* **Opción 3:** Capa de persistencia genérica (ORM) entre los servicios y la base de datos.

## Decisión Adoptada
Opción elegida: "**Opción 1: Acceso directo desde los servicios mediante el componente de acceso de PostgreSQL**", porque:
* **El DAO no aporta valor en este sistema:** como RD3 fija PostgreSQL, el DAO no sirve para aislar a los servicios de otro gestor de base de datos. Sus clases solo reenviarían las llamadas al componente de acceso, añadiendo una capa sin responsabilidad propia.
* **Simplifica el diseño:** hay una capa menos entre el negocio y los datos, y la dependencia de cada servicio con la base de datos queda explícita en el modelo (servicio → componente de acceso → PostgreSQL).
* **Usa el componente estándar de PostgreSQL:** `libpq` es el componente de acceso que describe la propia documentación de PostgreSQL y lo mantiene el mismo proyecto, por lo que no se introducen dependencias de terceros.
* **El tamaño del dominio no justifica más capas:** el sistema tiene pocos servicios y pocas entidades (`Suscriptor`, `EventoUso` y las noticias), así que el coste de una capa de abstracción supera su beneficio.

### Consecuencias
* **Positivas:** Arquitectura más simple, con menos componentes entre los servicios y la base de datos.
* **Negativas:** Los servicios pasan a conocer el acceso a la base de datos, que antes quedaba oculto tras los DAO, y quedan más acoplados a PostgreSQL (asumible, porque RD3 ya lo impone).
* **Neutras:** Desaparecen `AdminRedaccionDAO` y `SuscriptoresDAO` del modelo y aparece el componente de acceso. Las entidades `Suscriptor` y `EventoUso` se mantienen. Este ADR sustituye solo la parte de acceso a datos de los ADR-0001.1, 0003, 0005 y 0006.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Acceso directo mediante el componente de acceso de PostgreSQL
* **A favor (+):** Reduce el número de componentes del backend y las dependencias entre ellos: cada servicio se relaciona directamente con el componente de acceso.
* **A favor (+):** Usa el componente que documenta y mantiene el propio proyecto PostgreSQL, sin dependencias de terceros.
* **En contra (-):** Los servicios asumen la responsabilidad del acceso a datos y quedan más acoplados a PostgreSQL.

### Opción 2: Mantener la capa DAO
* **A favor (+):** Oculta el acceso a datos a los servicios y facilitaría sustituir el gestor de base de datos.
* **En contra (-):** Con PostgreSQL fijado por RD3, esa sustitución no es un caso real, y la capa se limita a reenviar llamadas al componente de acceso, añadiendo clases y dependencias sin aportar aislamiento útil.

### Opción 3: Capa de persistencia genérica (ORM)
* **A favor (+):** Los servicios trabajan con objetos de dominio y no con la base de datos.
* **En contra (-):** No es el componente que indica la documentación de PostgreSQL y añade una capa que el sistema no necesita.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar esta decisión en `Arquitecturas/`:
  1. Eliminar `AdminRedaccionDAO` y `SuscriptoresDAO` de `PkgDatos`.
  2. Añadir el componente de acceso de PostgreSQL y relacionar con él a `ServicioNoticias`, `ServicioEstadisticas`, `ServicioSuscripciones` y `ServicioFacturacion`.
* **Notas de Discusión ASS-ASC:**
  * *Aceptada tras la revisión de los arquitectos cognitivos.* Decisión originada en la indicación del profesor.
  * Al no poder consultar de nuevo al profesor, se toma como referencia la documentación oficial de PostgreSQL: su Parte IV ("Client Interfaces") presenta `libpq` como la interfaz cliente para conectarse al servidor y enviarle consultas, por lo que es el "componente de acceso" al que se refiere la indicación.

## Referencias
* PostgreSQL Global Development Group, "Part IV. Client Interfaces", [https://www.postgresql.org/docs/current/client-interfaces.html](https://www.postgresql.org/docs/current/client-interfaces.html)
* PostgreSQL Global Development Group, "libpq — C Library", [https://www.postgresql.org/docs/current/libpq.html](https://www.postgresql.org/docs/current/libpq.html)
