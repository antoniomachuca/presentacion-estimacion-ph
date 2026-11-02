# 0003 - Gestión de Cobros con Stripe Billing, Webhooks y Patrones Adapter y Observer

* **Estado:** aceptada
* **Fecha:** 2026-11-02
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
El periódico digital ofrece tres modalidades de suscripción de pago: trimestral (70 €), semestral (130 €) y anual (190 €). Los cobros deben realizarse a través de una pasarela externa, que según el catálogo de requisitos debe ser Stripe. En la arquitectura vigente (ADR-0001.3 y ADR-0002), el módulo de facturación del backend es el único que se comunica con Stripe mediante el canal HTTP/REST ya establecido, respetando el anonimato de las suscripciones y la segregación de datos (sin almacenar datos de tarjeta en la base de suscriptores, ver ADR-0004). La pregunta es cómo diseñar el cobro periódico de las tres modalidades y cómo se entera el sistema del resultado de cada cobro.

## Requisitos Motivadores (Decision Drivers)
* **Requisito(s) Clave:**
  * **RF4** - Cobro externalizado (El sistema debe realizar los cobros mediante una pasarela externa).
  * **RF3** - Modalidades de suscripción (Suscripciones trimestrales, semestrales y anuales con sus importes).
  * **RD1** - Uso de Stripe (La pasarela de pago utilizada debe ser Stripe).
* Restricciones del sistema: privacidad anónima (ADR-0004) y estilo Cliente-Servidor REST (RD2).

## Opciones Consideradas
* **Opción 1:** Stripe Billing (suscripciones recurrentes en Stripe) con notificación de resultados mediante Webhooks. El backend aísla Stripe con el patrón `Adapter` y reparte los resultados con el patrón `Observer`.
* **Opción 2:** Cron Job local en el backend que planifica los cobros e invoca la API de Stripe.
* **Opción 3:** Cola de mensajes (RabbitMQ) en el backend para encolar los cobros.

## Decisión Adoptada
Opción elegida: "**Opción 1: Stripe Billing con Webhooks, patrones `Adapter` y `Observer`**", porque:
* Cumple RF3, RF4 y RD1 de forma directa: la renovación periódica de cada modalidad la planifica Stripe a partir de los tres importes configurados en la pasarela, de modo que el backend no calcula ni dispara cobros.
* **Aísla Stripe mediante `Adapter`:** `ServicioFacturacion` depende de una interfaz propia y un `StripeAdapter` la traduce al lenguaje de Stripe. Así el resto del backend no conoce los tipos de Stripe y se reduce el acoplamiento con el proveedor.
* **Reparte los resultados mediante `Observer`:** el receptor de notificaciones de Stripe (`WebhookController`) actúa como sujeto y avisa a los observadores interesados (por ejemplo, `ServicioSuscripciones`, que actualiza el estado de pago de la suscripción). Añadir un nuevo interesado no obliga a cambiar el receptor.
* Encaja con el estilo Cliente-Servidor REST (RD2): Stripe notifica los resultados mediante HTTP a un endpoint del backend, sin añadir infraestructura nueva (ni planificador ni broker).
* Mantiene la privacidad: los datos de tarjeta los gestiona exclusivamente Stripe, y el sistema solo almacena identificadores opacos de Stripe vinculados al token anónimo del suscriptor.

### Consecuencias
* **Positivas:** Se elimina del backend la lógica de planificación de cobros; el estado del cobro tiene una única fuente de verdad (Stripe); menos componentes que modelar y desplegar; los cobros continúan aunque el backend esté caído temporalmente; cambiar de pasarela solo afectaría al `Adapter`.
* **Negativas:** Dependencia fuerte de Stripe; el endpoint de notificaciones es una superficie de entrada pública que debe autenticar a Stripe y tolerar eventos duplicados o desordenados; el estado local es eventualmente consistente con el de Stripe.
* **Neutras:** Los ASJ deben modelar la recepción de notificaciones de Stripe como una dependencia en sentido inverso, de Stripe hacia el backend. Esta recepción no discrimina dispositivo (RF1), por lo que no necesita pasar por el API Gateway.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Stripe Billing con Webhooks, patrones `Adapter` y `Observer`
* **A favor (+):** Delega en Stripe el ciclo de cobro de las tres modalidades (70 €, 130 € y 190 €), evitando programar planificadores y bloqueos contra cobros duplicados en `ServicioFacturacion`.
* **A favor (+):** Mantiene una única fuente de verdad financiera en la pasarela sin almacenar datos de método de pago en `SuscriptoresDAO` (únicamente identificadores opacos y el estado de la suscripción).
* **A favor (+):** El `Adapter` limita el *vendor lock-in* a una sola clase y el `Observer` mantiene desacoplado el receptor de notificaciones de quien las consume.
* **En contra (-):** Introduce consistencia eventual con la base de datos local: si el endpoint de webhooks sufre retrasos o caídas, el estado en `SuscriptoresDAO` queda desfasado frente al cobro real, además de exigir verificar la firma de Stripe y tratar eventos duplicados o desordenados.

### Opción 2: Cron Job local en el backend
* **A favor (+):** Control directo desde `ServicioFacturacion` sobre el calendario de cobros y mayor facilidad para cambiar de pasarela en el futuro.
* **A favor (+):** Actualización síncrona del estado de la suscripción en `SuscriptoresDAO` al recibir la respuesta de la API, sin exponer un endpoint público adicional.
* **En contra (-):** Obliga a persistir fechas de próximo cobro en `SuscriptoresDAO` y requiere bloqueos de concurrencia para evitar cobros dobles si hay varias instancias del backend.
* **En contra (-):** Reimplementa lógica crítica que Stripe ya ofrece de forma nativa (control de renovación), aumentando el riesgo de inconsistencias ante caídas del servidor durante el ciclo de facturación.

### Opción 3: Cola de mensajes (RabbitMQ)
* **A favor (+):** Desacopla las peticiones de cobro de las peticiones HTTP del periódico y conserva las pendientes si el servicio de facturación se reinicia.
* **En contra (-):** Introduce un broker de mensajería desproporcionado para el alcance del sistema (mismo criterio de contención aplicado en ADR-0001.3 al descartar microservicios por eventos).
* **En contra (-):** Por sí sola no determina cuándo vence cada periodo (trimestral, semestral o anual): sigue necesitando un planificador que consulte `SuscriptoresDAO` e inyecte el cobro en la cola.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar esta decisión en `Arquitecturas/`, ampliando el diagrama de la semana 2 con:
  1. La interfaz `PasarelaPago` (`<<interface>>`) usada por `ServicioFacturacion` y la clase `StripeAdapter` que la implementa y se comunica con la pasarela Stripe.
  2. El `WebhookController` como sujeto (`Subject`) que notifica a los observadores (`ServicioSuscripciones`), con la dependencia de Stripe hacia el backend.
  3. El estado de pago de la suscripción dentro del almacén de suscriptores.
* **Notas de Discusión ASS-ASC:**
  * Pendiente de contraste con Sergio González y Sara Guillen (ASC): debe completarse tras la sesión de revisión con las objeciones planteadas y la respuesta técnica de los ASS. Puntos previstos a discutir: sobreingeniería de los webhooks frente al cron y gestión de eventos duplicados o desordenados.
  * En la revisión, Sergio González (ASC) planteó si hacía falta utilizar webhooks o si el backend podía encargarse de los cobros mediante una tarea programada. Así se evitaría añadir un endpoint para recibir notificaciones de Stripe.
  * Sara Guillén (ASC) comentó que los webhooks podían llegar repetidos o en un orden distinto al esperado y que eso podía hacer que el estado guardado en la base de datos no coincidiera con el de Stripe.
  * Antonio Machuca y Enrique Aldama (ASS) explicaron que, con una tarea programada, el backend tendría que controlar los vencimientos. Con Stripe Billing esa parte queda en Stripe y los webhooks sirven para comunicar al sistema cómo ha terminado cada pago. También señalaron que habría que comprobar las notificaciones y evitar procesar un mismo evento más de una vez.
  * Después de comparar las opciones el equipo decidió mantener Stripe Billing con webhooks. Se consideró que era la opción más adecuada para gestionar los cobros periódicos, siempre que el backend controle correctamente las notificaciones recibidas.

## Referencias
* Stripe, "Billing: Subscriptions overview", [https://docs.stripe.com/billing/subscriptions/overview](https://docs.stripe.com/billing/subscriptions/overview)
* Stripe, "Receive events in your webhook endpoint", [https://docs.stripe.com/webhooks](https://docs.stripe.com/webhooks)
* Erich Gamma et al., *Design Patterns: Elements of Reusable Object-Oriented Software* (Patrones `Adapter` y `Observer`).
* Refactoring Guru, "Adapter Pattern", [https://refactoring.guru/design-patterns/adapter](https://refactoring.guru/design-patterns/adapter)
* Refactoring Guru, "Observer Pattern", [https://refactoring.guru/design-patterns/observer](https://refactoring.guru/design-patterns/observer)
* Chris Richardson, "Pattern: Idempotent Consumer", [https://microservices.io/patterns/communication-style/idempotent-consumer.html](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
