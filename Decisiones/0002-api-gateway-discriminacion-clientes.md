# 0002 - Incorporación de API Gateway para Discriminación de Clientes (PC y Tablet)

* **Estado:** aceptada
* **Fecha:** 2026-11-02
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
El periódico digital debe atender a lectores que acceden mediante terminales PC y Tablet. El requisito RF1 exige que el sistema discrimine el tipo de cliente conectado. En la arquitectura Cliente-Servidor con API REST (ADR-0001.3), las peticiones HTTP/REST llegan directamente al servidor. 

Para resolver este problema sin sobrecargar los controladores REST de negocio con inspección de cabeceras, el profesor Rafael Capilla nos ha guiado hacia la incorporación de un **API Gateway** perimetral como punto de entrada centralizado. La pregunta técnica a resolver es: ¿dónde y cómo discriminar clientes PC y Tablet manteniendo la arquitectura simple y desacoplada?

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF1 - Discriminación del cliente (El sistema discrimina si el cliente es un PC o una tablet).
* Centralizar el punto de entrada HTTP/REST para los clientes responsivos PC y Tablet.
* Desacoplar la inspección de cabeceras del código de negocio en el backend.

## Opciones Consideradas
* **Opción 1:** API Gateway perimetral como punto único de entrada e inspección de cliente.
* **Opción 2:** Middleware / Interceptor embebido en los controladores REST del backend.
* **Opción 3:** Detección exclusiva en Frontend (Client-Side) mediante CSS/JS sin componente servidor.

## Decisión Adoptada
Opción elegida: "**Opción 1: API Gateway perimetral como punto único de entrada e inspección de cliente**", porque:
* Cumple estrictamente el requisito RF1 centralizando la discriminación de PC y Tablet mediante la inspección de la cabecera HTTP `User-Agent` en el perímetro del sistema.
* Sigue la recomendación técnica directa del profesor Rafael Capilla, evitando acoplar responsabilidades de infraestructura a los controladores de negocio (`NoticiasController`, `SuscripcionesController`).
* Normaliza la cabecera del dispositivo inyectando un encabezado estándar (`X-Device-Type: PC | TABLET`) hacia el backend y hacia el subsistema de métricas (RF6).

### Consecuencias
* **Positivas:** Controladores REST limpios de lógica de inspección de terminales; punto único de entrada para routing; facilita la medición de métricas de acceso por dispositivo.
* **Negativas:** Introduce un componente adicional en la arquitectura que debe ser modelado y mantenido.
* **Neutras:** Los ASJ deben modelar el componente dentro del paquete `PkgApiGateway` entre la capa de presentación (`FrontendWeb`) y los servicios del backend.

## Pros y Contras de las Opciones Consideradas

### Opción 1: API Gateway perimetral como punto único de entrada e inspección de cliente
* **A favor (+):** Cumple RF1 en un único punto perimetral antes de transferir la petición al backend REST.
* **A favor (+):** Centraliza la discriminación en un único componente de entrada, evitando replicar comprobaciones de cabeceras en los controladores de noticias, suscripciones y facturación.
* **A favor (+):** Permite reenviar el tipo de dispositivo normalizado a los módulos de noticias, suscripciones y estadísticas.
* **En contra (-):** Requiere modelar un componente intermediario adicional en el diagrama de paquetes UML.

### Opción 2: Middleware / Interceptor embebido en los controladores REST del backend
* **A favor (+):** No añade un componente de red o despliegue independiente.
* **En contra (-):** Acopla la lógica de detección de clientes al proceso del backend de negocio.
* **En contra (-):** Si se añade un nuevo servicio REST en el futuro, se debe asegurar que aplique el interceptor manualmente.

### Opción 3: Detección exclusiva en Frontend (Client-Side) mediante CSS/JS
* **A favor (+):** Muy simple de implementar en el navegador sin componentes adicionales en el servidor.
* **En contra (-):** Incumple el requisito RF1 en el servidor, ya que el sistema backend no tiene constancia del dispositivo del lector.
* **En contra (-):** Impide que el módulo de estadísticas de uso (RF6) registre y analice la procedencia real de las lecturas.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben incluir en `REPO/Arquitecturas/` el paquete `PkgApiGateway` con el componente `ApiGateway` y la clase `DeviceDiscriminator`, situado entre el paquete `FrontendWeb` y los controladores del `BackendServidor`.
* **Notas de Discusión ASS-ASC:**
  * *Objeción de Sergio González (ASC):* Sergio cuestionó si un API Gateway representaba sobreingeniería para un periódico digital.
  * *Respuesta técnica ASS (Antonio Machuca y Enrique Aldama) con Sara Guillen (ASC):* Se argumentó que el profesor Rafael Capilla orientó expresamente al equipo a utilizar este patrón para desacoplar la discriminación de terminales del backend. Para evitar sobreingeniería (KISS), el API Gateway solo realizará dos funciones: discriminar el dispositivo (PC/Tablet) y redirigir la petición HTTP/REST al controlador correspondiente.

## Referencias
* Chris Richardson, "Pattern: API Gateway / Backends for Frontends", [https://microservices.io/patterns/apigateway.html](https://microservices.io/patterns/apigateway.html)
* Martin Fowler, "Gateway", [https://martinfowler.com/articles/gateway-pattern.html](https://martinfowler.com/articles/gateway-pattern.html)
* Prof. Rafael Capilla, Material docente y sesiones de tutoría técnica DAS, Universidad Rey Juan Carlos (2026).
* Documentación oficial DAS, "Anexo-P1-2026-Enero.pdf: Requisito RF1 (Discriminación de terminales PC y Tablet)".
