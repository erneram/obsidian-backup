---
title: Kubernetes
tags: [infrastructure, kubernetes, k8s, orquestacion, devops]
---

# ☸️ Kubernetes (K8s)

---

## ¿Qué es?

Kubernetes es una plataforma de **orquestación de contenedores** de código abierto, originalmente desarrollada por Google. Su trabajo es gestionar el ciclo de vida de contenedores en producción: desplegarlos, escalarlos, mantenerlos vivos y actualizarlos sin downtime.

Trabaja sobre un clúster de máquinas (nodos) y toma decisiones automáticas sobre dónde y cómo corren tus contenedores.

---

## ¿Cómo funciona?

### Arquitectura del clúster

```
┌────────────────────────────────────────────┐
│              Control Plane                 │
│  API Server │ Scheduler │ etcd │ Controller│
└──────────────────┬─────────────────────────┘
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
  Node 1        Node 2        Node 3
  [Pod A]       [Pod B]       [Pod C]
  [Pod D]       [Pod E]       [Pod F]
```

**Control Plane** — el cerebro del clúster:
- **API Server**: punto de entrada para toda comunicación (`kubectl` habla con él).
- **Scheduler**: decide en qué nodo corre cada Pod según recursos disponibles.
- **etcd**: base de datos distribuida que guarda el estado del clúster.
- **Controller Manager**: reconcilia el estado actual con el estado deseado.

**Nodes (Workers)** — las máquinas que ejecutan los contenedores:
- **kubelet**: agente que corre en cada nodo y gestiona los Pods.
- **kube-proxy**: maneja el networking entre Pods y servicios.

### Objetos clave

| Objeto | Descripción |
|--------|-------------|
| **Pod** | La unidad mínima. Uno o más contenedores que comparten red y almacenamiento |
| **Deployment** | Gestiona réplicas de Pods y rolling updates |
| **Service** | Expone Pods con una IP estable (load balancer interno) |
| **Ingress** | Enruta tráfico HTTP/HTTPS externo a los Services |
| **ConfigMap** | Variables de configuración no sensibles |
| **Secret** | Variables sensibles (contraseñas, tokens) cifradas en base64 |
| **Namespace** | Aislamiento lógico de recursos dentro del clúster |
| **PersistentVolume** | Almacenamiento persistente que sobrevive al Pod |

---

## ¿Qué hace?

- **Self-healing**: reinicia contenedores que fallan, reemplaza nodos caídos.
- **Escalado automático**: añade/quita réplicas según carga (HPA - Horizontal Pod Autoscaler).
- **Rolling updates**: despliega nueva versión sin downtime, hace rollback si algo falla.
- **Load balancing**: distribuye el tráfico entre réplicas automáticamente.
- **Gestión de secretos**: inyecta credenciales a los contenedores de forma segura.
- **Service discovery**: los Pods se encuentran entre sí por nombre sin IPs hardcodeadas.

---

## Configuración para un proyecto nuevo

### 1. `deployment.yaml` — define la app y sus réplicas

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
  namespace: production
spec:
  replicas: 3                      # 3 instancias corriendo en paralelo
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
        - name: laravel
          image: tu-registry/laravel-app:v1.2.0
          ports:
            - containerPort: 9000
          env:
            - name: APP_ENV
              value: production
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db-password
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
```

### 2. `service.yaml` — expone la app internamente

```yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
  namespace: production
spec:
  selector:
    app: laravel
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9000
  type: ClusterIP
```

### 3. `ingress.yaml` — enruta tráfico externo

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: miapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: laravel-service
                port:
                  number: 80
```

### 4. `secret.yaml` — credenciales seguras

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  db-password: c2VjcmV0MTIz   # base64: echo -n 'secret123' | base64
```

### 5. Comandos esenciales

```bash
# Aplicar todos los manifiestos de una carpeta
kubectl apply -f ./k8s/

# Ver estado de los Pods
kubectl get pods -n production

# Ver logs de un Pod
kubectl logs -f pod/laravel-app-xyz -n production

# Escalar manualmente
kubectl scale deployment laravel-app --replicas=5 -n production

# Hacer rollout de nueva imagen
kubectl set image deployment/laravel-app laravel=tu-registry/laravel-app:v1.3.0 -n production

# Ver historial y hacer rollback
kubectl rollout history deployment/laravel-app -n production
kubectl rollout undo deployment/laravel-app -n production
```

---

## ✅ Cuándo usar Kubernetes

| Escenario | Por qué K8s gana |
|-----------|-----------------|
| Alta disponibilidad requerida | Auto-healing y múltiples réplicas sin intervención manual |
| Tráfico variable / picos impredecibles | HPA escala pods automáticamente según CPU/memoria |
| Múltiples microservicios | Gestiona decenas de servicios con service discovery nativo |
| Zero-downtime deployments | Rolling updates y rollback automático integrados |
| Equipo grande con múltiples ambientes | Namespaces separan dev/staging/production en el mismo clúster |
| Multi-cloud o portabilidad | Los manifiestos YAML corren en cualquier cloud (EKS, GKE, AKS) |

> **No es la mejor opción** si tu app es un monolito pequeño en un solo servidor, o si el equipo no tiene experiencia en K8s — la curva de aprendizaje y la complejidad operacional pueden ser mayores que el beneficio.

---

## ↩️ Navegación
- [[Server/IaC/IaC|🏗️ IaC Hub]] → [[Server/Server|🖥️ Server]] → [[HOME|🏠 HOME]]
