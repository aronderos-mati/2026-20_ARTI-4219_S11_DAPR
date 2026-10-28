# analisis-cloudevents.md
# Completa este archivo con tus respuestas al TODO 4 de la Fase 3.

## TODO 4: Análisis de Trazabilidad (CloudEvents)

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

**`id`**: 
    **¿Quién lo genera?** Este identificador (ej. `8a8fe053-0f12-4326-a931-6c3e2749eff5`) lo genera automáticamente el sidecar de Dapr del servicio publicador en el momento en que intercepta los datos de la aplicación y los envuelve en el estándar CloudEvent (a menos que la aplicación lo asigne explícitamente antes).

   **¿Por qué es importante para la idempotencia?** Los brokers de mensajería (como Redis) generalmente garantizan una entrega de "al menos una vez" (*at-least-once*), lo que significa que por fallos de red podrían entregar el mismo mensaje dos veces. El consumidor puede guardar este `id` en una base de datos; si recibe un mensaje con un `id` que ya procesó, lo ignora. Esto asegura que la operación de negocio no se duplique (idempotencia).

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

**`source`**: 
   **¿Qué valor tiene?** En el log capturado, tiene el valor `order-publisher`.
   
   **¿De dónde lo toma Dapr?** Dapr lo toma de la identidad del servicio que originó el mensaje. Específicamente, lee la anotación `dapr.io/app-id` (o el argumento `--app-id`) del Pod del publicador y lo inyecta aquí.

### 3. El atributo `traceparent` y su relación con OpenTelemetry

**`traceparent`**: 
   **¿Qué estándar implementa?** Implementa el estándar **W3C Trace Context**.
   
   **¿Cómo se relaciona con OpenTelemetry?** OpenTelemetry utiliza este mismo estándar W3C como su formato nativo para propagar contextos distribuidos. Dapr inyecta este encabezado para que los colectores de OpenTelemetry y herramientas de observabilidad (como Jaeger o Zipkin) puedan unir gráficamente toda la historia: saber exactamente cuándo salió el mensaje del publicador, cuánto tiempo estuvo en el broker y cuándo llegó al suscriptor, viéndolo todo como una sola transacción continua.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents

 **Valor de negocio (Auditoría de 24h)**: 
   Para un equipo de auditoría, los atributos del sobre de CloudEvents que resultarían indispensables son:

   **`time`** (intrínseco al estándar): Permite acotar y filtrar de forma inmutable la búsqueda a los eventos generados estrictamente en la ventana de las últimas 24 horas.

   **`source`**: Responde a la pregunta "¿Quién lo hizo?", identificando de manera irrefutable qué microservicio o sistema originó la acción.

   **`type`**: Responde a la pregunta "¿Qué pasó?", permitiendo filtrar por acciones de negocio específicas (ej. si fue una creación, actualización o eliminación de orden).
   
   **`id` y/o `traceparent`**: Responden a la pregunta "¿Qué camino tomó?", permitiendo rastrear un evento específico a través de toda la red de microservicios para auditar si todos los sistemas reaccionaron correctamente.
