# 0006 - Reintentos de Pago Fallido mediante Patrón Retry

* **Estado:** aceptada
* **Fecha:** 2026-11-09
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
Con el cobro externalizado en Stripe Billing (ADR-0003), a veces un pago periódico falla y el sistema debe permitir reintentarlo hasta tres veces. La pregunta es quién ejecuta los reintentos y cómo se aplica una política de reintento (número máximo de intentos, espera entre ellos y qué hacer al agotarlos) sin guardar datos de tarjeta (ADR-0004) y manteniendo el backend sin estado (ADR-0001.3).

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF5 - Reintentos de pago (Cuando falla un pago, el sistema debe permitir reintentarlo hasta tres veces).
* Restricciones del sistema: cobro con Stripe y notificación mediante webhooks (ADR-0003), anonimato de las suscripciones (ADR-0004), backend sin estado (ADR-0001.3) y ausencia de infraestructura nueva.

## Opciones Consideradas
* **Opción 1:** Patrón `Retry` con la política de reintento delegada en Stripe y el backend reaccionando a los eventos que recibe.
* **Opción 2:** Patrón `Retry` implementado en `ServicioFacturacion` con un contador de intentos persistido.
* **Opción 3:** Cola de mensajes (RabbitMQ) con espera diferida para encolar los reintentos.

## Decisión Adoptada
Opción elegida: "**Opción 1: Patrón `Retry` con la política delegada en Stripe**", porque:
* Cumple RF5: la política de `Retry` se configura en Stripe Billing con un máximo de tres reintentos tras un fallo, con espera entre intentos, y el backend no necesita temporizadores propios.
* **Aplica `Retry` sin estado en el backend:** el número de intento y la fecha del siguiente reintento viajan en los eventos de Stripe (por ejemplo, `invoice.payment_failed`), de modo que el contador vive en Stripe y no en el servidor. Cuando se agotan los intentos, Stripe lo notifica y el backend marca la suscripción como suspendida.
* Reutiliza el `Observer` definido en ADR-0003: `ServicioSuscripciones` recibe los resultados de pago del `WebhookController` y actualiza la situación de pago de la suscripción.
* No añade infraestructura: no hacen falta planificador ni broker.

### Consecuencias
* **Positivas:** Se elimina del backend la lógica de temporización y de conteo de intentos; el backend sigue sin estado de sesión y puede escalar horizontalmente; la política (máximo de intentos y esperas) se cambia por configuración.
* **Negativas:** Dependencia de la política de reintentos de Stripe; la situación de pago local puede ir con retraso respecto a la de Stripe si falla un webhook; el procesado de webhooks debe ser idempotente ante eventos duplicados.
* **Neutras:** Los ASJ deben modelar el flujo de reintento y mostrar que las actualizaciones se disparan desde las notificaciones de Stripe.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Patrón `Retry` con la política delegada en Stripe
* **A favor (+):** Stripe ejecuta los hasta 3 reintentos y evita programar contadores de intentos ni bloqueos contra cobros duplicados en `ServicioFacturacion`.
* **A favor (+):** El backend permanece sin estado: solo registra el resultado de cada evento recibido.
* **En contra (-):** El intervalo entre reintentos depende de lo que ofrezca Stripe y el backend no lo controla directamente.
* **En contra (-):** Si el endpoint de webhooks falla, la situación local queda desfasada del cobro real.

### Opción 2: Patrón `Retry` en `ServicioFacturacion`
* **A favor (+):** Control total del número de intentos y de las esperas (por ejemplo, retroceso exponencial) entre los 3 reintentos.
* **A favor (+):** Actualización inmediata al recibir la respuesta de la API.
* **En contra (-):** Obliga a guardar el contador de intentos y la fecha del siguiente reintento en `SuscriptoresDAO`, y a evitar cobros dobles con varias instancias del backend.
* **En contra (-):** Reimplementa lógica que Stripe ya ofrece y aumenta el riesgo de inconsistencias si el servidor cae a mitad de un reintento.

### Opción 3: Cola de mensajes (RabbitMQ) con espera diferida
* **A favor (+):** Desacopla los reintentos de las peticiones HTTP y conserva los pendientes si el servicio de facturación se reinicia.
* **En contra (-):** Introduce un broker desproporcionado para el alcance del sistema (mismo criterio de contención de ADR-0001.3).
* **En contra (-):** Solo programa el envío; sigue haciendo falta contar intentos y decidir cuándo se agotan.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar esta decisión en `Arquitecturas/`:
  1. Mostrar que `ServicioSuscripciones`, como observador del `WebhookController` (ADR-0003), actualiza la situación de pago de la suscripción (al corriente, pago fallido o suspendida).
  2. Incluir la situación de pago de la suscripción en el almacén de suscriptores, sin datos de tarjeta.
* **Notas de Discusión ASS-ASC:**
  * Pendiente de contraste con Sergio González y Sara Guillen (ASC): completar tras la sesión de revisión. Puntos previstos: dependencia de los reintentos de Stripe frente a un control propio, y coherencia de la situación local ante eventos duplicados o desordenados.

## Referencias
* Stripe, "Automatic retries (Smart Retries) for failed payments", [https://docs.stripe.com/billing/revenue-recovery/smart-retries](https://docs.stripe.com/billing/revenue-recovery/smart-retries)
* Microsoft, "Retry pattern", Azure Architecture Center, [https://learn.microsoft.com/en-us/azure/architecture/patterns/retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
* Michael T. Nygard, *Release It!* (patrones de estabilidad, entre ellos `Retry`).
