# EventHub Kubernetes Manifests

Manifiestos Kubernetes para desplegar EventHub en el clúster `k3d-eventhub`. Argo CD sincroniza la infraestructura compartida y los siete microservicios desde este repositorio.

> Esta guía describe lo que configuran los manifiestos actuales. El repositorio no crea el clúster k3d, no instala Argo CD, no construye ni publica imágenes y no contiene credenciales. Esos componentes deben estar disponibles o prepararse por separado.

## Contenido

```text
apps/
  infrastructure.yaml             # Application de Argo CD para la infraestructura
  eventhub-services.yaml          # ApplicationSet que descubre los servicios dev
infrastructure/
  kustomization.yaml              # Namespace, configuración y dependencias comunes
  namespace/
  config/
  mongodb/
  rabbitmq/
  zipkin/
  secrets/
services/
  <servicio>/
    base/                          # Deployment, Service y configuración base
    overlays/dev/                  # Kustomize para el entorno dev e imagen
```

## Arquitectura desplegada

Todos los componentes de la aplicación están en el namespace `eventhub`. Los servicios Kubernetes son `ClusterIP`: se comunican dentro del clúster y no quedan expuestos directamente a la red del equipo.

| Componente | Puerto interno | Uso / dependencias |
| --- | ---: | --- |
| `api-gateway` | 8000 | Entrada HTTP de la aplicación y enrutamiento a los microservicios |
| `config-service` | 8081 | Configuración por tenant; MongoDB |
| `auth-service` | 8082 | Autenticación; PostgreSQL en Neon |
| `event-service` | 8083 | Eventos y branding; MongoDB y AWS S3 |
| `booking-service` | 8084 | Reservas; PostgreSQL en Neon y RabbitMQ |
| `payment-service` | 8085 | Stripe Checkout y webhooks; PostgreSQL en Neon y RabbitMQ |
| `notification-service` | 8086 | Notificaciones; MongoDB, RabbitMQ, correo y AWS S3 |
| `mongodb` | 27017 | Persistencia MongoDB para configuración, eventos y notificaciones |
| `rabbitmq` | 5672 / 15672 | Mensajería AMQP / consola de administración |
| `zipkin` | 9411 | Interfaz de trazas distribuidas |

Notification Service limita la consulta de branding a Event Service a 2 segundos para conectar y 3 segundos para recibir respuesta. Si vence el timeout, continúa generando y enviando el ticket con branding genérico; los valores se configuran en `spring.cloud.openfeign.client.config.event-service` dentro de `application.yaml`.

MongoDB y RabbitMQ se despliegan como `StatefulSet` de una réplica y solicitan almacenamiento persistente (2 GiB y 1 GiB, respectivamente). Los manifiestos no instalan PostgreSQL: los servicios relacionales usan las bases de datos externas configuradas en Neon. Zipkin recibe las trazas indicadas por `eventhub-global-config`.

Los despliegues de los microservicios incluyen probes HTTP de salud y límites/solicitudes de recursos. La infraestructura declara un `Ingress` de clase `traefik` que envía las solicitudes con ruta `/` al `api-gateway:8000`; no define un hostname ni TLS. Para que el Ingress sea accesible desde el equipo, el clúster debe tener Traefik activo y exponer su puerto HTTP al host (por ejemplo, mediante el mapeo configurado al crear k3d). Si eso no está configurado, usa `kubectl port-forward`.

## Requisitos previos

- Docker Desktop en ejecución.
- `k3d` y `kubectl` instalados.
- Un clúster k3d llamado `eventhub` (contexto `k3d-eventhub`).
- Argo CD instalado en ese clúster, en el namespace `argocd`.
- Acceso de Argo CD al repositorio Git indicado en `apps/infrastructure.yaml` y `apps/eventhub-services.yaml`.
- Imágenes de contenedor publicadas y accesibles en Docker Hub con el nombre `juankxxd/eventhub-<servicio>`.
- Credenciales válidas para MongoDB, RabbitMQ, Neon, los secretos internos, SMTP y AWS S3, según los servicios que se quieran habilitar.

Los overlays `dev` seleccionan la etiqueta `latest`. El despliegue requiere que esas imágenes existan; este repositorio no las compila. Para versiones reproducibles, cambia los overlays para usar etiquetas inmutables y actualiza el repositorio GitOps.

## Preparar y levantar el entorno

Los ejemplos de esta sección usan PowerShell y se ejecutan desde la raíz de este repositorio.

### 1. Verificar Docker y el clúster

```powershell
docker ps
k3d cluster list
```

`docker ps` muestra los contenedores activos de Docker, incluidos los nodos del clúster k3d; no muestra cada Pod de Kubernetes como un contenedor independiente. `k3d cluster list` muestra los clústeres k3d disponibles.

Si el clúster `eventhub` ya existe pero está detenido, inícialo y selecciona su contexto:

```powershell
k3d cluster start eventhub
kubectl config use-context k3d-eventhub
kubectl get nodes
```

`kubectl get nodes` confirma que Kubernetes ve los nodos. Si aún no existe el clúster, créalo primero con `k3d cluster create eventhub`; este repositorio no define opciones especiales de creación ni puertos publicados.

### 2. Comprobar Argo CD

```powershell
kubectl -n argocd get pods
```

Espera a que los Pods de Argo CD estén listos. Argo CD es un requisito externo: los archivos de `apps/` son recursos `Application` y `ApplicationSet`, no una instalación de Argo CD.

Para abrir la consola desde el equipo, deja este comando ejecutándose en una terminal:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8443:443
```

Abre `https://localhost:8443`. El certificado puede ser autofirmado en un entorno local. Para recuperar la contraseña inicial del usuario `admin` en PowerShell:

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

En Bash, la alternativa equivalente es:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

El Secret inicial puede dejar de existir después de completar la configuración de Argo CD. Cambia la contraseña inicial y no la guardes en el repositorio.

### 3. Crear el Secret de EventHub

Los Deployments esperan un Secret de Kubernetes llamado `eventhub-global-secrets`. El Secret no se crea desde los manifiestos GitOps: hay que aprovisionarlo en el clúster antes de sincronizar los componentes que lo consumen.

1. Copia el archivo de ejemplo y completa sus valores localmente:

   ```powershell
   Copy-Item infrastructure\secrets\secrets.example.env infrastructure\secrets\secrets.env
   ```

   Edita `infrastructure\secrets\secrets.env` con las credenciales reales. El archivo está excluido por `.gitignore`; no lo subas al repositorio ni pegues sus valores en tickets o logs.

2. Crea el namespace y el Secret:

   ```powershell
   kubectl apply -f infrastructure\namespace\namespace.yaml
   kubectl -n eventhub create secret generic eventhub-global-secrets --from-env-file=infrastructure\secrets\secrets.env --dry-run=client -o yaml | kubectl apply -f -
   ```

   El segundo comando permite crear el Secret o actualizarlo si ya existía, sin imprimir los valores del archivo. Ejecútalo de nuevo después de cambiar credenciales.

El archivo de ejemplo enumera estas claves:

| Claves | Uso |
| --- | --- |
| `MONGO_ROOT_USER`, `MONGO_ROOT_PASS` | Usuario y contraseña root de MongoDB |
| `RABBITMQ_USER`, `RABBITMQ_PASS` | Usuario y contraseña de RabbitMQ |
| `GATEWAY_INTERNAL_SECRET`, `JWT_SECRET` | Comunicación interna entre gateway y servicios, y firma JWT |
| `PAYMENT_WEBHOOK_HMAC_SECRET` | Secreto opcional para el webhook HMAC interno heredado; no verifica Stripe |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | Creación de Checkout Sessions y verificación del endpoint webhook de Stripe |
| `NEON_DB_USER`, `NEON_DB_PASS` | Credenciales de PostgreSQL en Neon |
| `NEON_DB_AUTH_URL`, `NEON_DB_BOOKING_URL`, `NEON_DB_PAYMENT_URL` | URLs JDBC de las bases de auth, booking y payment |
| `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD` | Configuración SMTP de notificaciones |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Acceso S3 usado por eventos y notificaciones |
| `GRAFANA_ADMIN_PASS` | Contraseña administrativa de Grafana |

Configura los valores de acuerdo con el proveedor y el entorno que estés usando. Además, el overlay de `event-service` contiene configuración no secreta de S3 (región, bucket, URL pública y un endpoint inicialmente vacío); ajústala en `services\event-service\base\kustomization.yaml` si tu entorno requiere valores distintos. No uses credenciales de producción en un clúster de desarrollo.

Para habilitar Stripe Checkout, añade `STRIPE_SECRET_KEY` y `STRIPE_WEBHOOK_SECRET` al archivo local `infrastructure\secrets\secrets.env` y vuelve a aplicar el Secret. Configura las URLs de retorno no secretas `PAYMENT_CHECKOUT_SUCCESS_URL` y `PAYMENT_CHECKOUT_CANCEL_URL` en `infrastructure\config\global-config.yaml`; `payment-service` solo habilita Checkout cuando ambas están definidas. Registra en Stripe el endpoint `POST /api/v1/payments/webhook/stripe` con los eventos `checkout.session.completed`, `checkout.session.expired`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `payment_intent.succeeded`, `payment_intent.payment_failed` y `refund.updated`. El webhook usa `STRIPE_WEBHOOK_SECRET` (prefijo `whsec_`), no `PAYMENT_WEBHOOK_HMAC_SECRET`. Payment Service persiste cada intento con su Checkout Session y permite como máximo una sesión activa por reserva; reconcilia y cierra las sesiones tras un fallo o al expirar el hold de Booking (10 minutos), y solo permite otro intento mientras ese hold siga vigente. La migración Flyway V4 crea el registro de intentos y elimina la unicidad de `payments.booking_id`; en bases existentes, deja que Flyway aplique V4 durante el despliegue y no la marques como baseline si ya se ha aplicado V3. El Ingress incluido aún no configura hostname ni TLS; publica una URL HTTPS accesible desde Stripe antes de registrar el webhook.

V4 no puede asociar Checkout Sessions creadas por versiones anteriores porque sus IDs no se persistían. Antes de actualizar un entorno con tráfico, espera a que terminen los holds pendientes y cierra manualmente en Stripe las sesiones antiguas que sigan abiertas; no habilites nuevos pagos hasta completar esa limpieza. La reconciliación automática de sesiones nuevas se ejecuta cada 15 segundos por defecto.

Booking Service declara dead-letter exchanges y colas para los consumidores de eventos de pago, incluidos los resultados de reembolso; Payment Service consume solicitudes de cancelación desde su propia cola con reintentos y DLQ. Notification Service hace lo mismo para `notification.booking.confirmed.queue`, cuya DLQ es `notification.booking.confirmed.dlq`. Notification reintenta hasta tres veces con backoff exponencial antes de enviar mensajes fallidos a la DLQ y registra en MongoDB los `messageId` ya procesados; para mensajes históricos sin `messageId`, deduplica por `bookingId`. El envío SMTP es al menos una vez: una caída después de que SMTP acepte el correo y antes de registrar el inbox puede causar un correo duplicado en el reintento. Las nuevas colas de reembolso y cancelación usan argumentos DLX al declararse. Si una cola existente se declara con argumentos diferentes, la nueva declaración no es compatible: antes de actualizar cada servicio, detén consumidores, drena o conserva los mensajes pendientes según el procedimiento operativo y recrea la cola con los argumentos DLX. No borres colas con mensajes pendientes sin una decisión explícita de recuperación.

### 4. Registrar las aplicaciones de Argo CD

Los recursos `Application` y `ApplicationSet` apuntan por defecto a la rama `main` de `https://github.com/eventhub-dev-environment/eventhub-k8s-manifests.git`. Si usas un fork o una URL diferente, actualiza `repoURL` y la rama en ambos archivos antes de aplicarlos. Argo CD lee la versión publicada en Git, por lo que los cambios locales deben estar en la rama configurada para que Argo los sincronice.

Aplica primero la Application de infraestructura y luego el ApplicationSet de servicios:

```powershell
kubectl apply -f apps\infrastructure.yaml
kubectl apply -f apps\eventhub-services.yaml
```

La Application `eventhub-infrastructure` sincroniza `infrastructure/`: configuración global, MongoDB, RabbitMQ, Zipkin, Prometheus y Grafana. Prometheus descubre los Pods del namespace `eventhub` con permisos limitados de lectura. El ApplicationSet `eventhub-services` detecta cada directorio `services/*/overlays/dev` y crea una Application por servicio, con sincronización automática, autorreparación y eliminación de recursos que ya no estén declarados (`prune`).

Comprueba la sincronización y el arranque:

```powershell
kubectl -n argocd get applications
kubectl -n eventhub get pods
kubectl -n eventhub get services
kubectl -n eventhub get pvc
```

Es normal que los Pods tarden mientras Kubernetes descarga las imágenes e inicia las dependencias. Para esperar a que una aplicación quede saludable en Kubernetes:

```powershell
kubectl -n eventhub rollout status deployment/api-gateway
```

Los componentes de datos (`mongodb` y `rabbitmq`) son `StatefulSet`; puedes comprobarlos con `kubectl -n eventhub get statefulsets`.

### 5. Acceder al API Gateway

Con Traefik funcionando y su puerto HTTP publicado desde el clúster, el Ingress enruta las solicitudes HTTP de `/` al API Gateway. Como no define hostname, TLS ni un puerto del host, la URL concreta depende del mapeo de puertos de k3d; este repositorio no crea ese mapeo.

La alternativa local que no depende del Ingress es dejar activo este reenvío en una terminal separada:

```powershell
kubectl -n eventhub port-forward svc/api-gateway 8000:8000
```

El gateway queda disponible desde el equipo en `http://localhost:8000`. Mantén abierta la terminal mientras lo uses; `Ctrl+C` detiene el reenvío, no el servicio del clúster. Los microservicios no se publican directamente y las solicitudes de aplicación deben pasar por el gateway.

También puedes abrir interfaces de administración locales con reenvíos temporales:

```powershell
kubectl -n eventhub port-forward svc/zipkin 9411:9411
kubectl -n eventhub port-forward svc/rabbitmq 15672:15672
kubectl -n eventhub port-forward svc/prometheus 9090:9090
kubectl -n eventhub port-forward svc/grafana 3000:3000
```

Con cada comando activo, Zipkin se consulta en `http://localhost:9411`, RabbitMQ Management en `http://localhost:15672`, Prometheus en `http://localhost:9090` y Grafana en `http://localhost:3000`. Inicia sesión en RabbitMQ con `RABBITMQ_USER` y `RABBITMQ_PASS`; Grafana usa el usuario `admin` y `GRAFANA_ADMIN_PASS`.

## Diagnóstico y operación

| Comprobación | Comando | Qué permite revisar |
| --- | --- | --- |
| Contenedores del clúster | `docker ps` | Estado de los contenedores de nodos k3d |
| Clústeres k3d | `k3d cluster list` | Si `eventhub` existe y está iniciado |
| Nodos Kubernetes | `kubectl get nodes` | Estado de los nodos (`Ready`) |
| Pods de Argo CD | `kubectl -n argocd get pods` | Estado del controlador GitOps |
| Aplicaciones Argo CD | `kubectl -n argocd get applications` | Salud y sincronización de infraestructura y servicios |
| Pods EventHub | `kubectl -n eventhub get pods` | Estado, reinicios y razones de espera |
| Servicios y PVC | `kubectl -n eventhub get services` / `kubectl -n eventhub get pvc` | Servicios internos y almacenamiento persistente |

Para investigar un Pod que no inicia:

```powershell
kubectl -n eventhub describe pod <nombre-del-pod>
kubectl -n eventhub logs deployment/<nombre-del-deployment>
kubectl -n eventhub get events --sort-by=.lastTimestamp
```

Sustituye los valores entre `<...>` por el nombre real. `describe` y `get events` ayudan a identificar, por ejemplo, Secrets faltantes, PVC sin enlazar o problemas de descarga de imágenes; `logs` muestra la salida del contenedor. Evita compartir logs sin revisar que no incluyan información sensible.

Problemas frecuentes:

- **`ImagePullBackOff`**: verifica que la imagen y la etiqueta configuradas en el overlay `dev` existan y que el clúster pueda descargarlas.
- **`CreateContainerConfigError`** o referencias a claves faltantes: confirma que existe `eventhub-global-secrets` en `eventhub` y que contiene las claves requeridas.
- **PVC `Pending`**: comprueba que el clúster tenga una StorageClass disponible y revisa `kubectl -n eventhub describe pvc <nombre-del-pvc>`.
- **Aplicación `OutOfSync` o `Missing` en Argo CD**: comprueba que `repoURL`/rama sean accesibles y que los cambios estén publicados en Git.
- **Servicio en ejecución pero inaccesible desde el host**: confirma que el `port-forward` siga activo; los Services son internos `ClusterIP`, no puertos publicados directamente.
- **Ingress inaccesible**: verifica que el controlador Traefik esté activo, que el clúster publique el puerto HTTP al host y que `kubectl -n eventhub get ingress` muestre `eventhub-ingress`.

Para detener temporalmente el clúster conservando su configuración:

```powershell
k3d cluster stop eventhub
```

Y para volver a iniciarlo:

```powershell
k3d cluster start eventhub
kubectl config use-context k3d-eventhub
```

Eliminar el clúster (`k3d cluster delete eventhub`) es una operación destructiva: puede borrar los datos locales almacenados en sus volúmenes persistentes.

## Cambios de manifiestos

- Los recursos comunes se componen desde `infrastructure/kustomization.yaml`.
- Cada microservicio mantiene recursos base en `services/<servicio>/base/` y selecciona su imagen/namespace desde `services/<servicio>/overlays/dev/`.
- Para validar localmente un overlay antes de publicarlo, ejecuta `kubectl kustomize services\<servicio>\overlays\dev`; para infraestructura, `kubectl kustomize infrastructure`.
- Publica los cambios en la rama a la que apunta Argo CD. La sincronización automatizada puede eliminar recursos administrados que se retiren de los manifiestos por la opción `prune`.
- No agregues contraseñas, tokens ni archivos `.env` al control de versiones. Gestiona los secretos mediante un mecanismo seguro apropiado para tu equipo y entorno.
