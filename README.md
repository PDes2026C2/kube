# TusViajes - Kubernetes

Infraestructura declarativa (IaC) para desplegar los servicios de **TusViajes** en un clúster de Kubernetes, consumiendo las imágenes de [Docker Hub (rodribolanos)](https://hub.docker.com/u/rodribolanos).

---

## Arquitectura del Clúster

```mermaid
flowchart TD
    subgraph Host ["Máquina Host / Navegador"]
        Browser["Navegador Web\n(localhost:5173)"]
        BrowserAPI["Llamadas AJAX\n(localhost:8080)"]
        DBClient["DataGrip / DBeaver\n(localhost:5432)"]
    end

    subgraph K8s ["Clúster de Kubernetes"]
        subgraph IngressLayer ["Servicios Expuestos (LoadBalancer)"]
            FE_SVC["Service: frontend\nLoadBalancer :5173 -> :80"]
            BE_SVC["Service: backend\nLoadBalancer :8080 -> :8080"]
            PG_SVC["Service: postgres\nLoadBalancer :5432 -> :5432"]
        end

        subgraph PodLayer ["Pods (1 Pod por Servicio - Stateless / DB)"]
            FE_POD["Pod: frontend\n(Nginx + React SPA)"]
            BE_POD["Pod: backend\n(Spring Boot API)"]
            PG_POD["Pod: postgres\n(PostgreSQL 16)"]
            FL_POD["Pod: flights-api\n(Spring Boot Service)"]
        end

        subgraph InternalLayer ["Servicios Privados (ClusterIP)"]
            FL_SVC["Service: flights-api\nClusterIP :8081 -> :8081"]
        end
    end

    Browser --> FE_SVC --> FE_POD
    BrowserAPI --> BE_SVC --> BE_POD
    DBClient --> PG_SVC --> PG_POD

    BE_POD -- "DNS interno: http://flights-api:8081" --> FL_SVC --> FL_POD
    BE_POD -- "JDBC: postgres:5432" --> PG_SVC
    FL_POD -- "JDBC: postgres:5432" --> PG_SVC
```

### Tabla de Servicios y Puertos

| Servicio | Tipo de Service | Puerto Host | Puerto Contenedor | Acceso | Propósito |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **frontend** | `LoadBalancer` | `5173` | `80` | Público (Host) | Web SPA servida por Nginx. |
| **backend** | `LoadBalancer` | `8080` | `8080` | Público (Host) | API REST consumida por el bundle cliente del navegador. |
| **postgres** | `LoadBalancer` | `5432` | `5432` | Público (Host) | Base de datos accesible para DataGrip / DBeaver / tests locales. |
| **flights-api** | `ClusterIP` | *N/A* | `8081` | Interno (Clúster) | Microservicio privado de vuelos, solo consultado por el backend. |

---

## Estructura del Repositorio

```text
kube/
├── configmaps/
│   ├── app-configmap.yaml              # Variables no sensibles (URLs, timezone, baseline Flyway)
│   └── frontend-nginx-configmap.yaml   # Configuración Nginx (SPA try_files y proxy /api/)
├── secrets/
│   └── postgres-secret.yaml            # Credenciales de BD en base64
├── deployments/
│   ├── postgres-deployment.yaml        # Deployment de PostgreSQL 16
│   ├── flights-deployment.yaml         # Deployment de Flights API (imagePullPolicy: Always)
│   ├── backend-deployment.yaml         # Deployment del Backend (imagePullPolicy: Always)
│   └── frontend-deployment.yaml        # Deployment del Frontend (imagePullPolicy: Always)
├── services/
│   ├── postgres-service.yaml           # Service LoadBalancer :5432
│   ├── flights-service.yaml            # Service ClusterIP :8081 (privado)
│   ├── backend-service.yaml            # Service LoadBalancer :8080
│   └── frontend-service.yaml           # Service LoadBalancer :5173
├── .gitignore                          # Excluye binarios, temporales y documentación privada
└── README.md
```

---

## Despliegue y Ejecución

### Prerrequisitos
* Docker Desktop con soporte de **Kubernetes habilitado**.
* `kubectl` disponible en el path de la terminal.

### 1. Aplicar todos los manifiestos
Desde la raíz del repositorio `kube`:
```powershell
kubectl apply -R -f .
```

### 2. Verificar el estado de los recursos
```powershell
kubectl get pods,svc
```
Todos los pods deben reportar `1/1 Running`:
```text
NAME                               READY   STATUS    RESTARTS   AGE
backend-xxxxxxxxxx-xxxxx           1/1     Running   0          45s
flights-api-xxxxxxxxxx-xxxxx       1/1     Running   0          45s
frontend-xxxxxxxxxx-xxxxx          1/1     Running   0          45s
postgres-xxxxxxxxxx-xxxxx          1/1     Running   0          45s
```

### 3. Puntos de Acceso
* **Frontend Web:** [http://localhost:5173](http://localhost:5173)
* **Backend API REST:** [http://localhost:8080/api/countries](http://localhost:8080/api/countries)
* **Base de datos (DataGrip/DBeaver):** `localhost:5432`, base `postgres`, usuario `admin`, clave `password`.

---

## Actualización de Imágenes sin Downtime

Cuando se publica una nueva versión de una imagen en Docker Hub bajo el mismo tag (`:main`), se fuerza la descarga y el reinicio progresivo (Rolling Update) mediante:

```powershell
# Reiniciar un servicio específico (ej: backend)
kubectl rollout restart deployment/backend

# Reiniciar todos los servicios de aplicación
kubectl rollout restart deployment/backend deployment/frontend deployment/flights-api

# Seguir el estado del despliegue en vivo
kubectl rollout status deployment/backend
```

---

## Comandos Útiles

```powershell
# Ver logs en vivo por etiqueta de aplicación
kubectl logs -f -l app=backend
kubectl logs -f -l app=flights-api
kubectl logs -f -l app=frontend

# Desmantelar todos los recursos del clúster
kubectl delete -R -f .
```
