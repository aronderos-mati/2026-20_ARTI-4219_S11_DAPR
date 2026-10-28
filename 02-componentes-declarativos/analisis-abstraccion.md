# analisis-abstraccion.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 2.

## TODO 2: Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

Si la empresa decide migrar de Redis a Apache Kafka mañana, **solo habría que modificar el archivo YAML del componente** (o crear uno nuevo para Kafka y eliminar el de Redis). **El código Python del subscriber y del publisher no cambiaría en absoluto**. Dapr abstrae la infraestructura subyacente; las aplicaciones interactúan con la API de Dapr utilizando un estándar común, por lo que el sidecar es el único que necesita saber que el sistema de mensajería real cambió.

### 2. Inventario de 5 brokers soportados por Dapr

| Nombre del Broker | `spec.type` en el YAML |
| :--- | :--- |
| Apache Kafka | `pubsub.kafka` |
| RabbitMQ | `pubsub.rabbitmq` |
| Azure Service Bus | `pubsub.azure.servicebus` |
| Google Cloud (GCP) Pub/Sub | `pubsub.gcp.pubsub` |
| AWS SNS/SQS | `pubsub.aws.snssqs` |

### 3. Manejo seguro de secretos en producción

Dapr ofrece el componente **Secret Store** (Almacén de secretos) para gestionar información sensible. Esto permite conectar a Dapr con gestores seguros como Kubernetes Secrets, HashiCorp Vault, AWS Secrets Manager o Azure Key Vault. En el manifiesto YAML del componente, en lugar de utilizar el campo `value` para asignar la contraseña en texto plano, se utiliza **`secretKeyRef`**. Esto le indica al componente de Dapr que debe ir a buscar el valor real al Secret Store configurado referenciando el nombre y la clave del secreto, manteniendo el YAML seguro para almacenarse en un repositorio Git.
