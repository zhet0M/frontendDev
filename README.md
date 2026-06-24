# 🚀 Innovatech — Frontend Despacho Dashboard

> Aplicación web para gestión de ventas y despachos, desplegada en AWS ECS con pipeline CI/CD automatizado.

![Deploy](https://img.shields.io/badge/Deploy-AWS%20ECS%20Fargate-orange?logo=amazon-aws)
![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-blue?logo=github-actions)
![Tech](https://img.shields.io/badge/Tech-React%20%2B%20Vite%20%2B%20Tailwind-61DAFB?logo=react)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📋 Descripción

Dashboard web desarrollado en React que permite a los usuarios de Innovatech Chile consultar órdenes de compra, gestionar despachos y visualizar el estado de los envíos. Se comunica con dos microservicios backend (Ventas y Despachos) desplegados en AWS ECS a través de un Application Load Balancer interno.

---

## 🏗️ Arquitectura de Despliegue

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│  Frontend ECS Task (subred pública)     │
│  React + Nginx · Puerto 8080            │
│  innovatech-frontend-svc                │
└───────────────┬─────────────────────────┘
                │ Variables de entorno (ALB DNS)
                ▼
┌─────────────────────────────────────────┐
│  ALB Interno · innovatech-internal-alb  │
│  :8080 → ventas-tg                      │
│  :8081 → despachos-tg                   │
└──────────┬──────────────┬───────────────┘
           │              │
           ▼              ▼
    ┌──────────┐   ┌──────────────┐
    │  Ventas  │   │  Despachos   │
    │  :8080   │   │    :8081     │
    └──────────┘   └──────────────┘
```

---

## 🛠️ Tecnologías

| Tecnología | Versión | Uso |
|---|---|---|
| React | 18+ | Framework UI |
| Vite | 5+ | Bundler y dev server |
| Tailwind CSS | 3+ | Estilos utilitarios |
| Nginx | 1.27-alpine | Servidor de archivos estáticos + proxy reverso |
| Docker | — | Contenedorización |
| AWS ECS Fargate | — | Orquestación en la nube |
| Amazon ECR | — | Registro de imágenes Docker |
| GitHub Actions | — | Pipeline CI/CD |

---

## 📁 Estructura del Proyecto

```
frontendDev/
├── public/                  # Archivos estáticos públicos
├── src/                     # Código fuente React
│   ├── components/          # Componentes reutilizables
│   ├── pages/               # Vistas principales
│   └── main.jsx             # Punto de entrada
├── nginx/
│   └── default.conf.template  # Configuración Nginx con proxy reverso
├── Dockerfile               # Imagen Docker multi-stage
├── docker-compose.yml       # Levantamiento local
├── index.html               # HTML raíz
├── vite.config.js           # Configuración Vite
├── tailwind.config.js       # Configuración Tailwind
└── package.json             # Dependencias Node.js
```

---

## ⚡ Ejecución Local

### Prerequisitos

- Node.js 18+
- Docker Desktop
- Backend corriendo localmente (ver repo `backendDev`)

### 1. Clonar el repositorio

```bash
git clone https://github.com/zhet0M/frontendDev.git
cd frontendDev
```

### 2. Instalar dependencias

```bash
npm install
```

### 3. Configurar variables de entorno

Crea un archivo `.env` en la raíz:

```env
BACKEND_VENTAS_URL=http://localhost:8080
BACKEND_DESPACHOS_URL=http://localhost:8081
```

### 4. Levantar con Docker Compose

```bash
docker compose up --build
```

La aplicación estará disponible en: **http://localhost:8080**

### 5. O en modo desarrollo (sin Docker)

```bash
npm run dev
```

---

## 🐳 Dockerfile (Multi-stage Build)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY nginx/default.conf.template /etc/nginx/templates/default.conf.template
ENV NGINX_ENVSUBST_FILTER="BACKEND_"
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```

El build multi-stage genera una imagen liviana (~30MB) que sirve los archivos estáticos compilados mediante Nginx, con proxy reverso hacia los backends configurado por variables de entorno.

---

## ☁️ Despliegue en AWS ECS

### Rama de despliegue

El pipeline CI/CD se activa automáticamente al hacer push a la rama `deploy`:

```bash
git checkout deploy
git merge main
git push origin deploy
```

### Pipeline CI/CD (GitHub Actions)

```
push → deploy branch
        │
        ▼
  1. Descargar repositorio (actions/checkout@v4)
        │
        ▼
  2. Configurar credenciales AWS (aws-actions/configure-aws-credentials@v4)
        │
        ▼
  3. Login ECR (aws-actions/amazon-ecr-login@v2)
        │
        ▼
  4. Definir nombre de imagen frontend (FRONTEND_IMAGE)
        │
        ▼
  5. Build y push Frontend (docker/build-push-action@v6)
        │
        ▼
  6. Deploy a ECS Frontend (aws ecs update-service)
        │
        ▼
  ✅ Nueva tarea ECS levantada
```

### GitHub Secrets requeridos

Configura los siguientes secrets en **Settings → Secrets and variables → Actions**:

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Clave de acceso AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta AWS |
| `AWS_SESSION_TOKEN` | Token de sesión temporal |
| `AWS_REGION` | Región (`us-east-1`) |
| `FRONTEND_INSTANCE_ID` | ID de instancia ECS (si aplica) |

### Variables de entorno en ECS (Task Definition)

```
BACKEND_VENTAS_URL    = http://internal-innovatech-internal-alb-[id].us-east-1.elb.amazonaws.com:8080
BACKEND_DESPACHOS_URL = http://internal-innovatech-internal-alb-[id].us-east-1.elb.amazonaws.com:8081
```

> ⚠️ Estas variables apuntan al DNS del ALB interno para evitar dependencias de IPs dinámicas de ECS.

---

## 🌐 Funcionalidades

| Funcionalidad | Descripción |
|---|---|
| 📦 Consultar órdenes de compra | Lista las últimas OC disponibles para despacho |
| 🚚 Generar orden de despacho | Crea una nueva orden de despacho asociada a una OC |
| 📋 Revisar órdenes de despacho | Consulta el estado de los despachos realizados |
| ✅ Cerrar despacho | Marca un despacho como entregado |

---

## 👥 Integrantes

- Cristóbal Medina
- Matías Medina

**Asignatura:** Introducción a Herramientas DevOps — ISY1101  
**Sección:** 004D | DuocUC 2025