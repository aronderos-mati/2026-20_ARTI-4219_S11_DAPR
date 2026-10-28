# reflexion-arquitectonica.md
# Completa este archivo con tus respuestas al Reto 3 de la Fase 4.

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

**Sin Dapr (SDK de Kafka directamente):** La configuración del `consumer group` y la lógica para gestionar la expiración (TTL) estarían fuertemente acopladas dentro del código fuente de la aplicación (por ejemplo, en variables de entorno, en un archivo de propiedades, o instanciadas directamente en el código del cliente). El responsable de mantenerlo sería el **equipo de la aplicación** (desarrollo), lo cual genera fricción, ya que deben lidiar con configuraciones de infraestructura en lugar de enfocarse en el negocio.

### 2. El patrón sidecar como frontera de responsabilidad

La distribución ideal separa el *qué* del *cómo*.
   **El equipo de producto (Desarrollo):** Es dueño de la regla de negocio y del SLA. Ellos definen el requerimiento ("una orden expira en 120 segundos").
   **El equipo de plataforma (Infraestructura/DevOps):** Es dueño de la implementación. Traducen ese requerimiento en manifiestos YAML (el Component de Dapr) y gestionan el broker. Esto permite aplicar políticas centralizadas sin modificar el código de los microservicios.

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

Aunque Dapr es agnóstico al broker y el código de la app no cambia, el manifiesto YAML sí puede requerir ajustes debido a las capacidades únicas de cada tecnología (un problema conocido como *abstracción con fugas*).
   **Limitación concreta con el TTL:** Redis (almacén en memoria) soporta expiración a nivel de *mensaje individual*. Sin embargo, Kafka (log inmutable) gestiona la retención de datos a nivel de *tópico* (basado en tiempo o tamaño). Al migrar a Kafka, el atributo `ttlInSeconds` a nivel de mensaje podría ser ignorado, obligando al equipo de plataforma a configurar la política de retención directamente en la infraestructura del clúster de Kafka.
