# proyecto_2_holdovers — El Quetzal en Kubernetes (k3s)

Namespace: `proyecto-2-holdovers` (los guiones bajos no son válidos en nombres de Kubernetes).
Todos los recursos llevan el sufijo `-holdovers`.

## Estructura de `k8s/`

| Archivo | Contenido |
|---|---|
| `00-namespace.yaml` | Namespace |
| `01-configmap.yaml` | ConfigMaps por servicio (hosts, puertos, nombres de BD, URLs entre servicios) |
| `02-secret.yaml.example` | Plantilla del Secret (el `02-secret.yaml` real **no** va al repo) |
| `03-postgres-init-cm.yaml` | ConfigMap con los scripts de `backend/db-init` (bases, esquemas y semillas) |
| `10-postgres.yaml` | PVC + Deployment + Service de PostgreSQL |
| `20` a `24-*.yaml` | Deployments del backend: catalogo, inventario, clientes, pedidos, reportes |
| `30-frontend.yaml` | Deployment + Service del frontend |
| `40-gateway.yaml` | Gateway nginx (ConfigMap + Deployment) y Services NodePort y LoadBalancer |
| `41-services.yaml` | Services ClusterIP del backend y del frontend |

## 0. Antes de empezar (en la VM)
```bash
kubectl config current-context        # debe ser el k3s de la VM, no docker-desktop
kubectl get nodes                     # Ready
kubectl get storageclass              # debe aparecer local-path (default)
docker compose down                   # libera el 8080 que usa el gateway de Compose
```

## 1. Imágenes
Los Deployments usan `gotzeth/<servicio>:1.0` (repositorio público en Docker Hub).
```bash
set -e
USER_HUB=gotzeth
TAG=${1:-1.0}

docker login
for svc in catalogo inventario clientes pedidos reportes; do
  IMG=$USER_HUB/$svc:$TAG
  docker build -t "$IMG" "backend/$svc"
  docker push "$IMG"
done

IMG=$USER_HUB/frontend:$TAG
docker build -t "$IMG" frontend      # usa frontend/Dockerfile (producción)
docker push "$IMG"
```
Si cambian el usuario o el tag, actualicen el campo `image:` de los Deployments (`20` a `24` y `30`).

## 2. Secret (no va al repo)
```bash
cp k8s/02-secret.yaml.example k8s/02-secret.yaml
nano k8s/02-secret.yaml               # poner contraseñas reales
git check-ignore -v k8s/02-secret.yaml   # debe mostrar la regla de .gitignore
```

## 3. Desplegar
Aplicar en este orden. Los Services (`41`) deben existir antes que el gateway (`40`): nginx resuelve los nombres al arrancar y falla con `host not found in upstream` si falta alguno.

```bash
# Base: namespace, configuración y secreto
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-configmap.yaml
kubectl apply -f k8s/02-secret.yaml
kubectl apply -f k8s/03-postgres-init-cm.yaml

# Base de datos (PVC + Deployment + Service)
kubectl apply -f k8s/10-postgres.yaml
kubectl rollout status deploy/postgres-holdovers -n proyecto-2-holdovers --timeout=180s

# Services internos
kubectl apply -f k8s/41-services.yaml

# Backend
kubectl apply -f k8s/20-catalogo.yaml
kubectl apply -f k8s/21-inventario.yaml
kubectl apply -f k8s/22-clientes.yaml
kubectl apply -f k8s/23-pedidos.yaml
kubectl apply -f k8s/24-reportes.yaml

# Frontend y gateway al final
kubectl apply -f k8s/30-frontend.yaml
kubectl apply -f k8s/40-gateway.yaml

# Esperar a que todo esté listo
kubectl rollout status deploy --timeout=180s -n proyecto-2-holdovers
```

> Los scripts de `docker-entrypoint-initdb.d` solo se ejecutan cuando el volumen de Postgres está vacío (primera vez). Si hay que reiniciar la base desde cero, ver la sección de solución de problemas.

## 4. Verificación del despliegue
Cada captura de evidencia debe mostrar el reloj del sistema y el carné. Anteponer `date;` al comando ayuda.

```bash
date; kubectl get nodes
date; kubectl get all -n proyecto-2-holdovers
kubectl get pods -n proyecto-2-holdovers              # todos Running y READY completo
kubectl get configmap,secret,pvc -n proyecto-2-holdovers
```

## 5. Verificación por criterio de la lista de cotejo

### A. Manifiestos y namespace (criterios 1 y 2)
```bash
date; kubectl get all -n proyecto-2-holdovers
kubectl get deploy -n proyecto-2-holdovers -o wide    # columna IMAGES: imágenes propias de Docker Hub
kubectl get pods -n proyecto-2-holdovers
```

### B. Persistencia (criterios 3 y 4)
```bash
kubectl get pvc -n proyecto-2-holdovers               # postgres-pvc en estado Bound
kubectl describe pod -l app=postgres-holdovers -n proyecto-2-holdovers | grep -A3 -i "volumes\|mounts"
```
Demostración de que el dato persiste:
```bash
# 1. En el navegador, crear un registro (por ejemplo un cliente o producto) con el carné
# 2. Borrar el Pod de la base de datos
date; kubectl delete pod -l app=postgres-holdovers -n proyecto-2-holdovers
# 3. Ver cómo el ReplicaSet lo recrea
kubectl get pods -n proyecto-2-holdovers -l app=postgres-holdovers -w     # Ctrl+C al estar Running
# 4. Recargar el navegador: el registro debe seguir ahí
```
No borrar el PVC después de esta prueba.

### C. Configuración desacoplada (criterios 5, 6 y 7)
```bash
# ConfigMap consumido por el Deployment
kubectl get configmap -n proyecto-2-holdovers
kubectl describe configmap catalogo-cfg-holdovers -n proyecto-2-holdovers
kubectl get deploy catalogo-holdovers -n proyecto-2-holdovers -o yaml | grep -B2 -A6 "configMapRef\|secretKeyRef"

# Secret: existe en el clúster y no en el repo (los valores salen en base64, no se muestran en claro)
kubectl get secret db-secret-holdovers -n proyecto-2-holdovers
git check-ignore -v k8s/02-secret.yaml
git ls-files k8s | grep -i secret                     # solo debe aparecer el .example

# El backend recibe las variables (no capturar DB_PASSWORD)
kubectl exec -n proyecto-2-holdovers deploy/catalogo-holdovers -- env | grep -E '^DB_(HOST|PORT|NAME|USER)='
```

### D. Ciclo de vida (criterios 8 y 9)
Escalado declarativo: editar `replicas` en el YAML y aplicar (no usar `kubectl scale`).
```bash
# Editar k8s/20-catalogo.yaml: replicas: 3
date; kubectl apply -f k8s/20-catalogo.yaml
kubectl get pods -n proyecto-2-holdovers -l app=catalogo
kubectl get deploy catalogo-holdovers -n proyecto-2-holdovers      # READY 3/3
```
Autoreparación:
```bash
kubectl get pods -n proyecto-2-holdovers -l app=catalogo
date; kubectl delete pod <nombre-de-un-pod> -n proyecto-2-holdovers
kubectl get pods -n proyecto-2-holdovers -l app=catalogo -w         # aparece uno nuevo
```
Al terminar, volver `replicas` al valor deseado y aplicar de nuevo.

### E. Exposición y gateway (criterios 10, 11 y 12)
```bash
# ClusterIP y endpoints no vacíos
kubectl get svc -n proyecto-2-holdovers
kubectl get endpoints -n proyecto-2-holdovers                       # ningún servicio sin endpoints

# NodePort (30080) y LoadBalancer (8080): usar la IP de la VM
kubectl get svc -n proyecto-2-holdovers                             # EXTERNAL-IP del gateway-lb-holdovers
curl http://<IP-VM>:30080/api/catalogo/health
curl http://<IP-VM>:8080/api/catalogo/health

# Rutas del gateway por nombre DNS del Service
curl http://<IP-VM>:8080/api/inventario/health
curl http://<IP-VM>:8080/api/clientes
curl http://<IP-VM>:8080/api/pedidos
curl "http://<IP-VM>:8080/api/reportes/<ruta>?desde=2026-01-01&hasta=2026-12-31"   # ajustar a la ruta real
curl -I http://<IP-VM>:8080/                                        # frontend

# Configuración del gateway
kubectl describe configmap gateway-nginx-conf-holdovers -n proyecto-2-holdovers
```
Respaldo si la red de la VM no deja llegar por NodePort/LoadBalancer. Usar un puerto local libre (el 8080 puede estar ocupado):
```bash
kubectl port-forward svc/gateway-nodeport-holdovers 9090:80 -n proyecto-2-holdovers
curl http://localhost:9090/api/catalogo/health
```

### F. Aplicación funcional (criterio 13)
Abrir `http://<IP-VM>:8080` en el navegador y ejecutar el CRUD completo (crear, consultar, actualizar, eliminar) en cada módulo, con la captura mostrando el reloj del sistema.

## 6. Captura final
```bash
date; kubectl get all -n proyecto-2-holdovers
```

## 7. Solución de problemas

| Síntoma | Qué revisar |
|---|---|
| `Endpoints` vacío | El `selector` del Service no coincide con los labels del Pod: `kubectl get pods --show-labels -n proyecto-2-holdovers` |
| Gateway en `CrashLoopBackOff` con `host not found in upstream` | Falta un Service o tiene otro nombre. Aplicar `41-services.yaml` y `30-frontend.yaml`, luego `kubectl rollout restart deploy/gateway-holdovers -n proyecto-2-holdovers` |
| `CreateContainerConfigError` | Falta el Secret o una clave: `kubectl describe pod <pod> -n proyecto-2-holdovers` |
| `ImagePullBackOff` | La imagen no existe en Docker Hub con ese tag |
| Postgres en `Pending` | PVC sin enlazar: `kubectl get pvc,storageclass`. El PVC debe usar el StorageClass por defecto del clúster |
| Backend responde 500 con `relation "..." does not exist` | Las bases están vacías: el ConfigMap `postgres-init-scripts` no tiene los esquemas o el volumen ya estaba inicializado (ver abajo) |
| Cambié un ConfigMap o Secret y no surte efecto | Se leen al crear el Pod: `kubectl rollout restart deploy/<nombre> -n proyecto-2-holdovers` |

Logs y eventos:
```bash
kubectl logs deploy/catalogo-holdovers -n proyecto-2-holdovers --tail=50
kubectl describe pod <pod> -n proyecto-2-holdovers
kubectl get events -n proyecto-2-holdovers --sort-by=.lastTimestamp
```

Reiniciar la base desde cero (**borra todos los datos**; el init solo corre con el volumen vacío). Borrar primero el Deployment y luego el PVC:
```bash
kubectl delete deploy postgres-holdovers -n proyecto-2-holdovers
kubectl delete pvc postgres-pvc -n proyecto-2-holdovers
kubectl apply -f k8s/03-postgres-init-cm.yaml -f k8s/10-postgres.yaml
kubectl logs deploy/postgres-holdovers -n proyecto-2-holdovers | grep -iE "error|CREATE"
kubectl exec -n proyecto-2-holdovers deploy/postgres-holdovers -- psql -U postgres -d catalogo_db -c '\dt'
```

## 8. Limpieza
```bash
kubectl delete namespace proyecto-2-holdovers     # borra todos los recursos, incluido el PVC y sus datos
```
