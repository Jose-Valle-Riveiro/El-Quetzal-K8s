# proyecto_2_holdovers — El Quetzal en Kubernetes (k3s)

Namespace: `proyecto-2-holdovers` (los guiones bajos no son válidos en nombres de Kubernetes).

## 0. Antes de empezar (en la VM)
```bash
kubectl config current-context        # debe ser el k3s de la VM, no docker-desktop
kubectl get nodes                     # Ready
docker compose down                   # libera el 8080 que usa el gateway de Compose
```

## 1. Imágenes
```bash
bash scripts/build-push.sh 1.0
```

## 2. Secret (no va al repo)
```bash
cp k8s/02-secret.yaml.example k8s/02-secret.yaml
nano k8s/02-secret.yaml               # poner contraseñas reales
```

## 3. Desplegar
```bash
kubectl apply -f k8s/00-namespace.yaml
bash scripts/generar-initdb-cm.sh     # crea k8s/03-postgres-init-cm.yaml
kubectl apply -f k8s/
kubectl get all -n proyecto-2-holdovers
```

## 4. Verificaciones por criterio
| Criterio | Comando |
|---|---|
| 1-2 Namespace / Pods Running | `kubectl get all -n proyecto-2-holdovers` |
| 3 PVC | `kubectl get pvc -n proyecto-2-holdovers` |
| 4 Persistencia | crear registro en la UI → `kubectl delete pod -n proyecto-2-holdovers -l app=postgres-holdovers` → `kubectl get pods -w` → refrescar UI |
| 5-7 ConfigMap/Secret/env | `kubectl get cm,secret -n proyecto-2-holdovers`; `kubectl exec -n proyecto-2-holdovers deploy/catalogo-holdovers -- env \| grep DB_` |
| 8 Escalado | editar `replicas: 3` en `k8s/20-catalogo.yaml` → `kubectl apply -f k8s/20-catalogo.yaml` |
| 9 Autoreparación | `kubectl delete pod -n proyecto-2-holdovers <pod-catalogo>` y `get pods -w` |
| 10 ClusterIP | `kubectl get endpoints -n proyecto-2-holdovers` (ninguno vacío) |
| 11 NodePort / LB | `http://<IP-VM>:30080` y `http://<IP-VM>:8080`; respaldo: `kubectl port-forward -n proyecto-2-holdovers svc/gateway-holdovers 8080:80` |
| 12 Gateway | `kubectl get cm gateway-conf-holdovers -n proyecto-2-holdovers -o yaml`; `curl -i http://<IP-VM>:30080/health` |
| 13 CRUD | desde el navegador |

Todas las capturas: reloj del sistema visible + carné.

## Notas
- La BD solo corre scripts de init con el volumen vacío. Si cambias los scripts: `kubectl delete -f k8s/10-postgres.yaml` (borra el PVC) y vuelve a aplicar.
- Para ver qué Pod respondió: `curl -i http://<IP-VM>:30080/health` muestra `X-Pod` y `X-Equipo`.
