# gitops-shop-demo — Infraestructura Kubernetes

Repositorio de infraestructura GitOps para deployment automatizado de microservicios usando ArgoCD y Helm.

**Principio fundamental:** Git es la unica fuente de verdad del estado del cluster.

ACLARACION: si no te interesa las cosas tecnicas y solo queres deployear, hay una carpeta dentro de gitops-shop-demo/scripts, la ejecutas y te deployea todo.

---

## Tabla de Contenidos

1. [Arquitectura](#arquitectura)
2. [Pre-requisitos](#pre-requisitos)
3. [Setup Completo desde Cero](#setup-completo-desde-cero)
4. [Estructura del Repositorio](#estructura-del-repositorio)
5. [Flujo de Deployment](#flujo-de-deployment)
6. [Como Probar un Cambio](#como-probar-un-cambio)
7. [Comandos Utiles](#comandos-utiles)
8. [Troubleshooting](#troubleshooting)
9. [Suposiciones y Simplificaciones](#suposiciones-y-simplificaciones)
10. [Repositorios Relacionados](#repositorios-relacionados)

---

## Arquitectura

### Flujo CI/CD Completo

El diagrama detallado de arquitectura esta disponible en el archivo `arquitectura.excalidraw`.

### Componentes

- **GitHub Actions (CI)**: Build, test, push Docker image, update infra repo
- **ArgoCD (CD)**: Monitorea Git, sincroniza cambios al cluster
- **Helm**: Templating y gestion de manifests Kubernetes
- **Kubernetes**: Cluster donde corren los microservicios

### Separacion de Responsabilidades

**CI (GitHub Actions)**:
1. Ejecuta tests
2. Construye Docker image con tag unico (TIMESTAMP-HASH)
3. Pushea imagen a DockerHub
4. Actualiza este repositorio (values.yaml)
5. NO toca el cluster directamente

**CD (ArgoCD)**:
1. Hace git pull cada 3 minutos
2. Detecta cambios en values.yaml
3. Renderiza templates con Helm
4. Aplica cambios al cluster
5. Es el UNICO que hace kubectl apply

**Principio clave**: CI actualiza Git, CD actualiza Cluster. Esta separacion asegura que Git sea la unica fuente de verdad.

---

## Pre-requisitos

- Docker Desktop o Docker Engine instalado
- Minikube instalado
- kubectl instalado
- Helm instalado
- Git configurado

---

## Setup Completo desde Cero

### Paso 1: Iniciar Minikube

```bash
minikube start --cpus=4 --memory=8192
```

### Paso 2: Crear Namespaces

```bash
kubectl create namespace dev
kubectl create namespace test
kubectl create namespace argocd
```

### Paso 3: Instalar ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que los pods esten listos
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Paso 4: Acceder a ArgoCD UI

```bash
# Obtener password inicial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# GUARDAR ESTE PASSWORD

# Port-forward para acceder a la UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Abrir navegador
# URL: https://localhost:8080
# Usuario: admin
# Password: (el que obtuviste arriba)
```

### Paso 5: Clonar Repositorio de Infraestructura

```bash
mkdir -p ~/Documentos/gitops-shop-demo
cd ~/Documentos/gitops-shop-demo

# Clonar este repositorio
git clone https://github.com/Gitops-Facundoh3/gitops-shop-demo.git
cd gitops-shop-demo
```

### Paso 6: Desplegar Applications en ArgoCD

```bash
# Aplicar todas las ArgoCD Applications
kubectl apply -f argocd-apps/backend-dev.yaml
kubectl apply -f argocd-apps/backend-test.yaml
kubectl apply -f argocd-apps/frontend-dev.yaml
kubectl apply -f argocd-apps/frontend-test.yaml

# Verificar Applications creadas
kubectl get applications -n argocd

# Expected output:
# NAME            SYNC STATUS   HEALTH STATUS
# backend-dev     Synced        Healthy
# backend-test    Synced        Healthy
# frontend-dev    Synced        Healthy
# frontend-test   Synced        Healthy
```

### Paso 7: Verificar Pods

```bash
# Ver pods en DEV
kubectl get pods -n dev

# Expected:
# NAME                                     READY   STATUS    RESTARTS   AGE
# backend-dev-backend-xxxxx-yyyyy          1/1     Running   0          2m
# frontend-dev-frontend-xxxxx-yyyyy        1/1     Running   0          2m

# Ver pods en TEST
kubectl get pods -n test

# Expected:
# NAME                                     READY   STATUS    RESTARTS   AGE
# backend-test-backend-xxxxx-yyyyy         1/1     Running   0          2m
# backend-test-backend-xxxxx-zzzzz         1/1     Running   0          2m
# frontend-test-frontend-xxxxx-aaaaa       1/1     Running   0          2m
# frontend-test-frontend-xxxxx-bbbbb       1/1     Running   0          2m
```

---

## Estructura del Repositorio

```
gitops-shop-demo/
├── argocd-apps/              # ArgoCD Applications
│   ├── backend-dev.yaml      # Backend DEV application
│   ├── backend-test.yaml     # Backend TEST application
│   ├── frontend-dev.yaml     # Frontend DEV application
│   └── frontend-test.yaml    # Frontend TEST application
│
├── helm-charts/              # Helm Charts
│   ├── backend/
│   │   ├── Chart.yaml        # Metadata del chart
│   │   ├── values-dev.yaml   # Valores para DEV
│   │   ├── values-test.yaml  # Valores para TEST
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   │
│   └── frontend/
│       ├── Chart.yaml
│       ├── values-dev.yaml
│       ├── values-test.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
│
├── arquitectura.excalidraw   # Diagrama de arquitectura
└── README.md
```

---

## Flujo de Deployment

### Descripcion General

El flujo completo desde codigo hasta cluster tarda aproximadamente 6-10 minutos.

### Paso a Paso Detallado

#### 1. Developer hace cambio en el codigo

```bash
# En repo de codigo (BackEnd-Shop o FrontEnd-Shop)
cd ~/Documentos/gitops-shop-demo/BackEnd-Shop

# Hacer cambios en el codigo
vim src/main/java/...

# Commit
git add .
git commit -m "feat: Add new feature"

# Crear tag (determina el ambiente de destino)
git tag v1.0-dev      # Para DEV
# o
git tag v1.0-test     # Para TEST

# Push del commit PRIMERO
git push origin master

# Push del tag DESPUES (SEPARADO)
git push origin v1.0-dev
```

**IMPORTANTE**: El tag debe pushearse por separado del commit para que GitHub Actions lo detecte correctamente.

#### 2. GitHub Actions (CI) ejecuta automaticamente

El pipeline detecta el tag y ejecuta:

1. **Build**: Compila la aplicacion (Maven o npm)
2. **Test**: Ejecuta tests unitarios
3. **Docker Build**: Construye imagen con tag unico `TIMESTAMP-HASH`
4. **Docker Push**: Sube imagen a DockerHub
5. **Update Infrastructure**: Clona este repositorio, actualiza `values-dev.yaml` o `values-test.yaml` segun el tag, hace commit y push

**Duracion**: 3-5 minutos

#### 3. ArgoCD detecta el cambio

- ArgoCD hace `git pull` cada 3 minutos
- Compara estado deseado (Git) vs estado actual (Cluster)
- Detecta que `values-dev.yaml` cambio

**Duracion**: 1-3 minutos

#### 4. ArgoCD sincroniza automaticamente

1. Ejecuta `helm template` para generar manifests
2. Compara manifests con estado del cluster
3. Detecta diferencia (OutOfSync)
4. Aplica cambios con `kubectl apply`
5. Kubernetes hace rolling update de los pods
6. ArgoCD valida health de los nuevos pods
7. Marca la Application como Synced y Healthy

**Duracion**: 1-2 minutos

### Logica de Ambientes

```bash
# Tags con sufijo -dev -> despliegan a namespace dev
git tag v1.0-dev

# Tags con sufijo -test -> despliegan a namespace test
git tag v1.0-test
```

### Diferencias entre DEV y TEST

| Caracteristica | DEV | TEST |
|---------------|-----|------|
| **Replicas Backend** | 1 | 2 |
| **Replicas Frontend** | 1 | 2 |
| **CPU Request** | 50m / 200m | 100m / 200m |
| **Memory Request** | 64Mi / 256Mi | 128Mi / 256Mi |
| **Namespace** | `dev` | `test` |

---

## Como Probar un Cambio

### Opcion 1: Deployment Manual

```bash
# 1. Ir al repositorio de codigo
cd ~/Documentos/gitops-shop-demo/BackEnd-Shop

# 2. Hacer un cambio visible
echo "# Test deployment $(date)" >> README.md

# 3. Commit y tag
git add README.md
git commit -m "test: Deployment to dev - $(date +%Y%m%d-%H%M%S)"
git tag v1.5-dev
git push origin master
git push origin v1.5-dev

# 4. Monitorear GitHub Actions
# Abrir: https://github.com/Gitops-Facundoh3/BackEnd-Shop/actions

# 5. Verificar que el repo de infra se actualizo
cd ~/Documentos/gitops-shop-demo/gitops-shop-demo
git pull
cat helm-charts/backend/values-dev.yaml | grep tag:

# 6. Monitorear ArgoCD
kubectl get application backend-dev -n argocd -w

# 7. Ver pods actualizandose
kubectl get pods -n dev -w
```

#### Frontend a TEST

```bash
cd ~/Documentos/gitops-shop-demo/FrontEnd-Shop

echo "# Test deployment $(date)" >> README.md
git add README.md
git commit -m "test: Frontend to test"
git tag v1.5-test
git push origin master
git push origin v1.5-test
```

### Opcion 2: Script Automatizado

```bash
chmod +x test-deployment.sh
./test-deployment.sh
```

### Verificacion Rapida Post-Deployment

```bash
cd ~/Documentos/gitops-shop-demo/gitops-shop-demo
git pull -q

echo "TAG EN VALUES:"
cat helm-charts/backend/values-dev.yaml | grep tag:

echo "IMAGEN EN PODS:"
kubectl describe pod -n dev -l app=backend | grep "Image:" | head -2

echo "ESTADO DE PODS:"
kubectl get pods -n dev -l app=backend
```

---

## Comandos Utiles

```bash
# Ver todas las applications ArgoCD
argocd app list

# Forzar sync manual
argocd app sync backend-dev

# Ver logs de un pod
kubectl logs -n dev -l app=backend --tail=50

# Ver eventos del namespace
kubectl get events -n dev --sort-by='.lastTimestamp'

# Reiniciar deployment
kubectl rollout restart deployment/backend-dev -n dev

# Ver estado detallado de una application
argocd app get backend-dev
```

---

## Troubleshooting

### Pod en CrashLoopBackOff

```bash
kubectl logs -n dev -l app=backend --previous
kubectl describe pod -n dev -l app=backend
```

### ArgoCD no sincroniza

```bash
# Ver estado
argocd app get backend-dev

# Forzar refresh
argocd app get backend-dev --refresh

# Forzar sync
argocd app sync backend-dev --force
```

### GitHub Actions falla

Verificar en la pestaña Actions del repositorio de codigo. Los errores mas comunes son:
- Secrets de DockerHub no configurados
- Token de GitHub sin permisos de escritura al repo de infra

---

## Suposiciones y Simplificaciones

- **Base de datos**: La aplicacion usa H2 en memoria para simplificar el demo. En produccion usaria PostgreSQL o MySQL con PersistentVolumes.
- **Secrets**: Los secrets de DockerHub y GitHub se configuran manualmente en los repositorios. En produccion se usaria un secret manager (AWS Secrets Manager, Vault).
- **TLS/HTTPS**: No implementado en este demo. En produccion se usaria cert-manager con Let's Encrypt.
- **Monitoreo**: No incluido. En produccion se agregaria Prometheus + Grafana.
- **Image Tags**: Tags basados en timestamp-hash. En produccion se usaria semantic versioning (v1.2.3).

---

## Decisiones Tecnicas Justificadas

### Por que separar repositorios de codigo e infraestructura?

1. **Separacion de concerns**: Developers se enfocan en codigo, Platform team en infra
2. **Permisos granulares**: Diferentes equipos, diferentes accesos
3. **Ciclo de vida diferente**: Codigo cambia frecuentemente, infra es mas estable
4. **Auditabilidad**: Cada cambio de infra tiene commit con autor y timestamp

### Por que Helm y no manifests planos?

1. **DRY**: Un template, multiples ambientes con values diferentes
2. **Rollback**: `helm rollback` vuelve a version anterior
3. **Community**: Miles de charts pre-hechos disponibles

### Por que ArgoCD y no Flux?

1. **UI intuitiva**: Visualizacion del estado del cluster en tiempo real
2. **Helm support**: Integrado de primera clase
3. **Adopcion**: CNCF graduated project, ampliamente usado en produccion

### Por que tags y no branches para determinar ambiente?

1. **Explicito**: `v1.0-dev` es claro, no ambiguo
2. **Auditable**: Git history muestra que se deployo y cuando
3. **Rollback facil**: Re-push tag viejo o revert commit

---

## Repositorios Relacionados

| Repositorio | Descripcion | URL |
|-------------|-------------|-----|
| **BackEnd-Shop** | Codigo Spring Boot | https://github.com/Gitops-Facundoh3/BackEnd-Shop |
| **FrontEnd-Shop** | Codigo React | https://github.com/Gitops-Facundoh3/FrontEnd-Shop |
| **gitops-shop-demo** | Infraestructura GitOps (este repo) | https://github.com/Gitops-Facundoh3/gitops-shop-demo |

---

**Preparado por:** Facundo Herrera
**Fecha:** 2025
