
# Products Launcher — NestJS Microservices

<p align="center">
  <a href="http://nestjs.com/" target="blank"><img src="https://nestjs.com/img/logo-small.svg" width="120" alt="Nest Logo" /></a>
</p>

Repositorio principal que orquesta todo el ecosistema de microservicios construido con [NestJS](https://nestjs.com/). Usa Git Submodules para gestionar cada servicio de forma independiente y Docker Compose para ejecutarlos en conjunto.

---

## Arquitectura

```
                    ┌─────────────────┐
          HTTP      │  client-gateway │   Puerto: 3000
  Cliente ─────────►│   (API Gateway) │
                    └────────┬────────┘
                             │ NATS
              ┌──────────────┼──────────────┐──────────────┐
              ▼              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
        │products  │  │ orders   │  │ payments │  │  auth    │
        │   -ms    │  │   -ms   │  │   -ms    │  │   -ms    │
        │(SQLite)  │  │(Postgres)│  │(Stripe)  │  │(Postgres)│
        └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### Microservicios

| Servicio | Puerto interno | Descripción |
|---|---|---|
| `client-gateway` | 3000 | API Gateway HTTP — punto de entrada único |
| `products-ms` | 3001 | CRUD de productos (SQLite + Prisma) |
| `orders-ms` | 3002 | Gestión de órdenes (PostgreSQL + Prisma) |
| `payments-ms` | 3003 | Pagos con Stripe + webhooks |
| `auth-ms` | 3004 | Autenticación y JWT (PostgreSQL + Prisma) |
| `nats-server` | 4222 / 8222 | Bus de mensajes entre microservicios |

---

## Inicio rápido (Dev)

### 1. Clonar el repositorio con sus submodules
```bash
git clone --recurse-submodules https://github.com/Nest-Microservicesa/products-launcher.git
cd products-launcher
```

Si ya clonaste sin `--recurse-submodules`:
```bash
git submodule update --init --recursive
```

### 2. Crear el archivo de variables de entorno
```bash
cp .env.template .env
```

Edita `.env` con tus valores:
```env
CLIENT_GATEWAY_PORT=3000
AUTH_DATABASE_URL=postgresql://postgres:password@localhost:5432/authdb?schema=public
PAYMENTS_MS_PORT=3003
STRIPE_SECRET=sk_test_xxxxxxxxxxxxxxxxxxxx
STRIPE_SUCCESS_URL=http://localhost:3000/payments/success
STRIPE_CANCEL_URL=http://localhost:3000/payments/cancel
STRIPE_ENDPOINT_SECRET=whsec_xxxxxxxxxxxxxxxxxxxx
```

### 3. Levantar todos los servicios con Docker
```bash
docker-compose up --build
```

---

## Gestión de Git Submodules

### Añadir un nuevo submodule
```bash
git submodule add <repository_url> <directory_name>
git add .
git commit -m "Add submodule"
git push
```

### Actualizar submodules al clonar
```bash
git submodule update --init --recursive
```

### Actualizar referencias a la última versión de cada submodule
```bash
git submodule update --remote
```

---

## ⚠️ Importante

Al trabajar con submodules, **siempre haz push en el submodule primero** y luego en el repositorio principal.

Si haces push en el repositorio principal antes que en el submodule, las referencias quedarán desactualizadas y tendrás que resolver conflictos.

---

## Licencia

MIT
