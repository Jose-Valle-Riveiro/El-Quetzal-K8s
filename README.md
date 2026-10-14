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
set -e
USER_HUB=<USUARIO_DE_HUB>
EQUIPO=holdovers
CARNE=<CARNE_DE_USUARIO_DE_HUB>
TAG=${1:-1.0}

docker login
for svc in catalogo inventario clientes pedidos reportes; do
  IMG=$USER_HUB/$EQUIPO-$svc-$CARNE:$TAG
  docker build -t "$IMG" "backend/$svc"
  docker push "$IMG"
done

IMG=$USER_HUB/$EQUIPO-frontend-$CARNE:$TAG
docker build -t "$IMG" frontend      # usa frontend/Dockerfile (producción)
docker push "$IMG"```

## 2. Secret (no va al repo)
```bash
cp k8s/02-secret.yaml.example k8s/02-secret.yaml
nano k8s/02-secret.yaml               # poner contraseñas reales
```

## 3. Desplegar
--COMPLETAR--
