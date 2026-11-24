# 2. Kubernetes: Pod, Deployment y Service

> **Pregunta típica:** "¿Qué es un Pod, un Deployment y un Service en Kubernetes? ¿Cómo se relacionan?"

> 🔗 **Relacionado:** [Docker: image vs container](01-docker-image-vs-container.md). Kubernetes **orquesta** los contenedores que salen de esas imágenes.

---

## Respuesta corta (para decir en entrevista)

> - Un **Pod** es la **unidad mínima** que Kubernetes despliega: uno o más contenedores que comparten **red** (misma IP, se hablan por `localhost`) y **volúmenes**. Normalmente hay un contenedor por Pod. Los Pods son **efímeros**: si mueren no se "reparan", se reemplazan por otros nuevos **con otra IP**.
> - Un **Deployment** es el objeto que **gestiona los Pods** de forma declarativa. Le digo "quiero 3 réplicas de la imagen `catalog-api:1.4`" y, a través de un **ReplicaSet**, se encarga de que siempre existan 3. Si un Pod muere, crea otro. También hace **rolling updates** (cambia la versión de a poco, sin caída) y **rollbacks**.
> - Un **Service** le da a ese conjunto cambiante de Pods una **dirección estable** (un nombre DNS y una IP virtual) y **balancea la carga** entre ellos. Elige a qué Pods enviar tráfico con un **selector de labels**.
>
> La relación es: el **Deployment crea y mantiene los Pods**, los Pods llevan **labels**, y el **Service los encuentra por esas labels** y les manda tráfico. Así los clientes nunca necesitan conocer las IPs de los Pods.

---

## Explicación a fondo

### Cómo se relacionan

```
                 Cliente / otro microservicio
                            │
                            ▼  http://catalog-api  (DNS estable)
                 ┌────────────────────────┐
                 │  Service catalog-api   │  selector: app=catalog-api
                 │  ClusterIP 10.96.12.4  │
                 └──────────┬─────────────┘
              balancea      │  (solo a Pods "Ready")
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │  Pod    │         │  Pod    │         │  Pod    │   labels: app=catalog-api
   │10.1.0.7 │         │10.1.2.3 │         │10.1.1.9 │   (IPs cambian todo el tiempo)
   └─────────┘         └─────────┘         └─────────┘
        ▲                   ▲                   ▲
        └───────── ReplicaSet (replicas: 3) ────┘
                            ▲
                 Deployment catalog-api  (image: catalog-api:1.4)
```

### Comparación

| | Pod | Deployment | Service |
|---|---|---|---|
| Qué es | 1+ contenedores que corren juntos | Controlador que mantiene N Pods iguales | Punto de acceso de red estable |
| Vida | Efímera | Persistente (estado deseado) | Persistente |
| ¿Tiene IP estable? | No | — | **Sí** (ClusterIP + DNS) |
| Se autorrecupera | No, por sí solo | **Sí** (recrea Pods) | — |
| Se crea a mano normalmente | Casi nunca | **Sí** | **Sí** |
| Analogía | Un trabajador | El jefe de turno que asegura que siempre haya 3 trabajadores | El número de teléfono de la oficina |

### Modelo declarativo y *reconciliation loop*

Kubernetes no ejecuta órdenes del tipo "crea un Pod". Recibe un **estado deseado** (YAML) y sus controladores comparan todo el tiempo **deseado vs actual**, y corrigen la diferencia. Por eso si borras un Pod de un Deployment, aparece otro al instante.

### Tipos de Service

| Tipo | Accesible desde | Uso |
|---|---|---|
| `ClusterIP` (por defecto) | Solo dentro del clúster | Comunicación entre microservicios |
| `NodePort` | Un puerto en cada nodo | Pruebas, on-premise simple |
| `LoadBalancer` | Internet, a través del load balancer de la nube (en AWS: un NLB/ELB) | Exponer un servicio directamente |
| `ExternalName` | — | Alias DNS a un servicio externo |

> En producción, el tráfico HTTP externo suele entrar por un **Ingress** (o Gateway API), que enruta por dominio/ruta hacia Services `ClusterIP`.

### Objetos relacionados que suelen preguntar

- **ReplicaSet**: lo crea el Deployment; casi nunca se toca directamente.
- **ConfigMap / Secret**: configuración y secretos inyectados como variables de entorno o archivos.
- **StatefulSet**: como Deployment pero con identidad estable, para bases de datos.
- **DaemonSet**: un Pod por nodo (agentes de logs, monitoreo).
- **HorizontalPodAutoscaler (HPA)**: ajusta el número de réplicas según CPU, memoria o métricas propias.
- **Namespace**: separación lógica (por ejemplo `staging` y `production`).

---

## Ejemplo de aplicación real

**Escenario:** desplegar la imagen `catalog-api` del tema de Docker en un clúster (por ejemplo **Amazon EKS**). Requisitos: 3 réplicas, actualizaciones sin caída, que otros microservicios la llamen por nombre y que se exponga hacia internet.

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: catalog-api            # ← qué Pods gestiona este Deployment
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0           # nunca bajar de 3 Pods listos durante un deploy
      maxSurge: 1                 # crear 1 Pod extra a la vez
  template:                       # ← plantilla de los Pods
    metadata:
      labels:
        app: catalog-api          # ← label que también usa el Service
    spec:
      containers:
        - name: api
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/catalog-api:1.4.2   # tag inmutable
          ports:
            - containerPort: 8080
          env:
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef: { name: catalog-db, key: connection-string }
          resources:
            requests: { cpu: 250m, memory: 256Mi }   # lo que el scheduler reserva
            limits:   { memory: 512Mi }              # si lo supera → OOMKilled
          readinessProbe:                            # ¿puede recibir tráfico?
            httpGet: { path: /health/ready, port: 8080 }
            periodSeconds: 5
          livenessProbe:                             # ¿está colgado? → reiniciar
            httpGet: { path: /health/live, port: 8080 }
            initialDelaySeconds: 10
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-api               # → DNS: catalog-api.production.svc.cluster.local
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: catalog-api              # ← envía tráfico a los Pods con esta label
  ports:
    - port: 80                    # puerto del Service
      targetPort: 8080            # puerto del contenedor
```

Otro microservicio, por ejemplo `orders-api`, simplemente llama a `http://catalog-api/products`, sin saber cuántos Pods hay ni sus IPs.

### Ingress (exposición a internet)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog
  namespace: production
spec:
  rules:
    - host: api.mitienda.com
      http:
        paths:
          - path: /products
            pathType: Prefix
            backend:
              service: { name: catalog-api, port: { number: 80 } }
```

### Qué pasa en un deploy de la versión 1.4.3

```bash
kubectl set image deployment/catalog-api api=.../catalog-api:1.4.3 -n production
kubectl rollout status deployment/catalog-api -n production
```

1. El Deployment crea un **nuevo ReplicaSet** con la versión 1.4.3.
2. Levanta 1 Pod nuevo (`maxSurge: 1`).
3. Cuando su **readinessProbe** pasa, el Service empieza a mandarle tráfico.
4. Recién entonces apaga 1 Pod viejo, y repite hasta completar las 3 réplicas.
5. Si la versión nueva nunca queda *ready*, el rollout se detiene y los Pods viejos siguen atendiendo. Para volver atrás:

```bash
kubectl rollout undo deployment/catalog-api -n production
```

**Resultado:** despliegues sin caída, autorrecuperación si un nodo muere, y escalado con `kubectl scale` o con un HPA en los picos de tráfico (por ejemplo, en un Black Friday).

---

## Comandos útiles para depurar

```bash
kubectl get pods -l app=catalog-api -o wide     # Pods, estado, IP y nodo
kubectl describe pod <pod>                      # eventos: ImagePullBackOff, OOMKilled, probes fallando
kubectl logs <pod> -f
kubectl get endpointslices -l kubernetes.io/service-name=catalog-api   # ¿el Service encontró Pods?
kubectl rollout history deployment/catalog-api
```

---

## Errores comunes / trampas

- **El selector del Service no coincide con las labels de los Pods** → el Service no tiene endpoints y responde con error de conexión. Es el error más típico.
- **Crear Pods sueltos** (sin Deployment) → si mueren, nadie los recrea.
- **Sin readinessProbe** → el Service manda tráfico a Pods que todavía están arrancando, y hay errores en cada deploy.
- **Confundir liveness con readiness**: una liveness que revisa la BD reinicia todos los Pods cuando la BD tiene un problema, y empeora la caída.
- **Sin `resources.requests`** → el scheduler no puede repartir bien los Pods y aparecen problemas de *noisy neighbor*.
- **Guardar archivos en el Pod** → se pierden. Usa volúmenes persistentes u object storage (S3).
- Pensar que un Pod = un contenedor: pueden ser varios (patrón *sidecar*: proxy, agente de logs).

---

## Preguntas de seguimiento para practicar

1. ¿Qué diferencia hay entre Deployment y ReplicaSet? ¿Y entre Deployment y StatefulSet?
2. ¿Cómo sabe el Service a qué Pods mandar tráfico? ¿Qué pasa si un Pod no está *ready*?
3. ¿Qué diferencia hay entre `readinessProbe` y `livenessProbe`?
4. ¿Cómo hace Kubernetes un rolling update sin caída? ¿Qué controlan `maxSurge` y `maxUnavailable`?
5. ¿Qué diferencia hay entre un Service `ClusterIP`, `NodePort` y `LoadBalancer`? ¿Y un Ingress?
6. ¿Por qué dos contenedores del mismo Pod pueden hablarse por `localhost`?
7. Un Pod está en `CrashLoopBackOff`. ¿Cómo lo investigas?
