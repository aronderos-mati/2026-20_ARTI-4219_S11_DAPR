# analisis-control-plane.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 1.

## TODO 1: Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

* **Tipo de objeto:** A nivel de carga de trabajo, el `dapr-sidecar-injector` se despliega como un **Deployment** (lo que genera el ReplicaSet y el Pod que ves en ejecución). A nivel de la API de Kubernetes, actúa como el backend para un **MutatingWebhookConfiguration**.
* **Por qué es necesario:** Kubernetes utiliza este mecanismo para interceptar y modificar recursos antes de que sean persistidos en `etcd`. El webhook instruye al API Server para que pause la creación de un Pod y envíe su manifiesto JSON al inyector. Esto le permite al inyector evaluar si el Pod tiene la anotación correspondiente, modificar el manifiesto "al vuelo" añadiendo el contenedor sidecar, y devolverlo al API Server para su creación final.

### 2. Réplicas del `dapr-operator` y comparación con Istio

* **Réplicas:** Según la salida de `kubectl get pods`, actualmente hay **1 sola réplica** del operador corriendo (`dapr-operator-cd69b76c7-qhr7v`).
* **Por qué es más liviano que Istio:** Istio es un *Service Mesh* enfocado en infraestructura y red. Su plano de control (Istiod) debe calcular, actualizar y sincronizar constantemente configuraciones complejas de enrutamiento (xDS), políticas y certificados a miles de proxies Envoy en tiempo real. Dapr, en cambio, es un *Application Mesh*. Su operador (`dapr-operator`) se limita a gestionar Custom Resource Definitions (CRDs) estáticos (como componentes de estado o pub/sub) y entregarlos a los sidecars solo cuando estos los solicitan. Al no tener que recalcular mapas de enrutamiento de red de bajo nivel, su consumo de CPU y memoria es drásticamente menor.

### 3. Flujo de inyección del sidecar

1. **Petición:** Ejecutas `kubectl apply` de un Deployment. El API Server de Kubernetes recibe la orden de crear los Pods.
2. **Intercepción (Admission Control):** Antes de persistir el Pod en `etcd`, el API Server verifica sus webhooks y envía el manifiesto original del Pod al servicio `dapr-sidecar-injector`.
3. **Evaluación:** El inyector revisa el manifiesto. Al detectar la anotación `dapr.io/enabled: "true"`, sabe que debe actuar. También lee otras anotaciones asociadas (como `dapr.io/app-id` o `dapr.io/app-port`).
4. **Mutación:** El inyector modifica el manifiesto JSON del Pod agregando el contenedor `daprd` a la lista. Utiliza sus propias variables de entorno (como `SIDECAR_IMAGE: ghcr.io/dapr/daprd:1.19.0`) para saber qué versión inyectar y añade los volúmenes de certificados gestionados por `dapr-sentry`.
5. **Persistencia:** El inyector devuelve el manifiesto mutado al API Server, el cual lo aprueba y lo guarda en `etcd`.
6. **Despliegue:** El `kubelet` del nodo asignado (ej. `dapr-lab-control-plane`) detecta la asignación y descarga tanto la imagen de la aplicación como la imagen del sidecar.
7. **Ejecución:** Ambos contenedores arrancan dentro del mismo Pod, compartiendo el mismo espacio de red, lo que permite que la aplicación y el sidecar de Dapr se comuniquen localmente a través de `localhost`.

---

## TODO 2: Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

Al observar los recursos instalados en el clúster, vemos que Dapr utiliza varios CRDs para gestionar su comportamiento. Cuatro de los más relevantes son:

* **`components.dapr.io`**: Define las configuraciones de los servicios de infraestructura externos que la aplicación va a usar, como bases de datos (State Stores), brokers de mensajería (Pub/Sub) o almacenes de secretos.
* **`configurations.dapr.io`**: Define políticas y configuraciones de comportamiento para los sidecars, tales como el nivel de trazabilidad distribuida (tracing), la recolección de métricas o las reglas de control de acceso entre servicios.
* **`subscriptions.dapr.io`**: Permite definir de forma declarativa a qué tópicos de eventos (mensajería Pub/Sub) está suscrita una aplicación, y a qué ruta o endpoint de la aplicación deben enviarse esos mensajes.
* **`resiliencies.dapr.io`**: Define las políticas de tolerancia a fallos, como la cantidad de reintentos (retries), tiempos máximos de espera (timeouts) y cortacircuitos (circuit breakers) para asegurar que las fallas temporales no tumben la aplicación.


### 5. Relación entre `components.dapr.io` y el concepto de Building Block

El **"Building Block"** (bloque de construcción) representa la **abstracción** (la API estándar). Por ejemplo, el bloque "State Management" (Gestión de Estado) le ofrece a tu código una forma genérica de guardar y recuperar datos, sin importar cómo o dónde se guarden realmente. Tu código siempre hace la misma llamada HTTP o gRPC.

El CRD **`components.dapr.io`** representa la **implementación concreta**. Es el manifiesto de configuración que le indica al sidecar de Dapr qué tecnología específica utilizar en un entorno determinado para respaldar esa abstracción (por ejemplo, usar *Redis* en tu máquina de desarrollo, pero usar *Azure CosmosDB* o *AWS DynamoDB* en producción).

En resumen: la aplicación programa contra el *Building Block* (la abstracción), mientras que el operador de Dapr configura el *Component* (la implementación concreta).
