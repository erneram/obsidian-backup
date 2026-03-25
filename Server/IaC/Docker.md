---
title: Docker
tags: [infrastructure, docker, contenedores, devops]
---

# 🐳 Docker

---

## ¿Qué es?

Docker es una plataforma de **contenedorización**. Permite empaquetar una aplicación junto con todas sus dependencias (librerías, runtime, variables de entorno, configuración) en una unidad portable llamada **contenedor**.

Un contenedor se comporta igual en tu máquina local, en el servidor de staging y en producción — elimina el clásico "en mi máquina funciona".

---

## ¿Cómo funciona?

Docker se apoya en dos conceptos principales:

### Imagen
Un **blueprint** inmutable que describe el sistema de archivos del contenedor. Se construye capa por capa a partir de un `Dockerfile`. Es como una "foto" del entorno de la app.

### Contenedor
Una **instancia en ejecución** de una imagen. Es el proceso real que corre tu aplicación. Puede iniciarse, detenerse y destruirse sin afectar la imagen original.

```
Dockerfile  ──build──►  Imagen  ──run──►  Contenedor (proceso vivo)
```

Docker usa características del kernel de Linux (`namespaces`, `cgroups`) para aislar cada contenedor del resto del sistema — sin necesitar una VM completa.

---

## ¿Qué hace?

- **Aísla** la app del sistema operativo host.
- **Estandariza** el entorno: misma versión de PHP/Node/Python en todos lados.
- **Acelera** el onboarding: un nuevo dev levanta el proyecto con un solo comando.
- **Facilita** el despliegue: la misma imagen que se probó en dev va a prod.
- **Compone** servicios (app + base de datos + cache) con Docker Compose.

---

## Configuración para un proyecto nuevo

### 1. `Dockerfile` (imagen de la app)

```dockerfile
# Ejemplo: Laravel con PHP 8.3 + FPM
FROM php:8.3-fpm-alpine

WORKDIR /var/www/html

# Instalar extensiones PHP necesarias
RUN docker-php-ext-install pdo pdo_mysql opcache

# Copiar dependencias primero (aprovecha caché de capas)
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader

# Copiar el resto del código
COPY . .

RUN chown -R www-data:www-data /var/www/html/storage

EXPOSE 9000
CMD ["php-fpm"]
```

### 2. `docker-compose.yml` (stack completo)

```yaml
version: "3.9"

services:
  app:
    build: .
    container_name: laravel_app
    volumes:
      - .:/var/www/html
    depends_on:
      - db
      - redis
    environment:
      APP_ENV: local
      DB_HOST: db
      REDIS_HOST: redis

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - .:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: app_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: root_secret
    volumes:
      - db_data:/var/lib/mysql

  redis:
    image: redis:alpine

volumes:
  db_data:
```

### 3. Comandos esenciales

```bash
# Construir y levantar todos los servicios
docker compose up --build -d

# Ver logs en tiempo real
docker compose logs -f app

# Ejecutar comando dentro del contenedor
docker compose exec app php artisan migrate

# Detener y eliminar contenedores
docker compose down

# Eliminar también los volúmenes (cuidado: borra datos)
docker compose down -v
```

---

## ✅ Cuándo usar Docker

| Escenario | Por qué Docker gana |
|-----------|---------------------|
| Múltiples devs en el equipo | Garantiza que todos tienen el mismo entorno |
| App con muchas dependencias (PHP, Node, Redis, DB) | Levanta todo el stack con un comando |
| Despliegue en diferentes clouds o VPS | La imagen corre igual en AWS, DigitalOcean o bare metal |
| CI/CD pipelines | Las builds son reproducibles y no dependen del runner |
| Microservicios | Cada servicio tiene su propio contenedor aislado |

> **No es la mejor opción** si tienes una app muy simple en un solo VPS y no necesitas portabilidad — un setup directo en el servidor puede ser suficiente.

---

## ↩️ Navegación
- [[Server/IaC/IaC|🏗️ IaC Hub]] → [[Server/Server|🖥️ Server]] → [[HOME|🏠 HOME]]
