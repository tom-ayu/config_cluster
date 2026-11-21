# Taller Kubernetes: Deployment y ReplicaSet con Excalidraw

Despliegue de **Excalidraw** (pizarra web, aplicación sin estado) en un clúster minikube, aplicando ReplicaSet, Deployment, namespace con cuota, exposición (NodePort e Ingress con TLS) y autoescalado (HPA).

## Entorno

- minikube (driver Docker) en Windows, kubectl
- Imagen: `excalidraw/excalidraw:latest`
- Namespace: `lab-excalidraw`
- Addons: `metrics-server`, `ingress`

## Manifiestos

| Archivo | Qué define |
|---|---|
| `01-namespace.yaml` | Namespace `lab-excalidraw` |
| `02-quota.yaml` | ResourceQuota (8 pods, CPU y memoria acotadas) |
| `03-replicaset.yaml` | ReplicaSet independiente de 2 pods (demo) |
| `04-deployment.yaml` | Deployment con RollingUpdate (`maxSurge: 1`, `maxUnavailable: 0`), readiness probe, requests y limits |
| `05a-service-nodeport.yaml` | Exposición **insegura**: NodePort, HTTP sin cifrado |
| `05b-service-clusterip.yaml` | Service interno ClusterIP (detrás del Ingress) |
| `06-ingress.yaml` | Ingress con TLS (`excalidraw.local`) |
| `07-hpa.yaml` | HorizontalPodAutoscaler (2 a 5 réplicas, CPU) |
| `08-carga.yaml` | Generador de carga para probar el HPA |

## Orden de ejecución

```bash
minikube start --driver=docker
minikube addons enable metrics-server
minikube addons enable ingress

kubectl apply -f 01-namespace.yaml -f 02-quota.yaml
kubectl apply -f 03-replicaset.yaml          # demo; luego borrar el ReplicaSet
kubectl apply -f 04-deployment.yaml
kubectl apply -f 05a-service-nodeport.yaml   # exposición insegura (demo)
kubectl apply -f 05b-service-clusterip.yaml  # reemplaza al NodePort
kubectl apply -f 06-ingress.yaml
kubectl apply -f 07-hpa.yaml
kubectl apply -f 08-carga.yaml               # genera carga para el HPA
```

**Secret TLS (no incluido en el repo).** Antes del Ingress, crear el certificado autofirmado y el Secret:

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 30 -keyout tls.key -out tls.crt -subj "/CN=excalidraw.local"
kubectl create secret tls excalidraw-tls --key tls.key --cert tls.crt -n lab-excalidraw
```

`tls.key` y `tls.crt` están en `.gitignore`. Para acceder: agregar `127.0.0.1 excalidraw.local` al archivo `hosts` y ejecutar `minikube tunnel`.

## Evidencias

| Captura | Qué demuestra |
|---|---|
| c1, c2 | Clúster activo, namespace y cuota |
| c3, c4 | ReplicaSet: auto-recuperación y no actualiza pods existentes |
| c5, c6 | Deployment, ReplicaSet y pods; configuración (estrategia, probe, recursos) |
| c7, c8, c9 | Escalado, rolling update y rollback |
| c10 | Exposición con NodePort (HTTP) |
| c11 | Exposición con Ingress y TLS (ClusterIP) |
| c12, c13 | HPA en reposo y escalando bajo carga |
| c15 | Cuota agotada: Deployment 8/10 por exceder el límite de pods |

## Limpieza

```bash
kubectl delete namespace lab-excalidraw
```