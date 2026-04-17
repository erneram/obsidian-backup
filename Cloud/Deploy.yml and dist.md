Aquí tienes un `.md` completo que documenta **qué subir a S3** y **cómo configurar tu deploy en GitHub Actions** 👇

---

# 🚀 Guía: Deploy de Frontend a S3 + CloudFront

Esta guía cubre:

- 📦 Qué archivos subir a S3
    
- ⚙️ Configuración correcta del pipeline (GitHub Actions)
    
- ⚡ Buenas prácticas de cache y performance
    

---

# 📦 1. ¿Qué debo subir a S3?

## ❌ NO hacer esto

- Subir `.zip`
    
- Subir la carpeta `dist` completa
    

Ejemplo incorrecto:

```
dist/index.html
dist/assets/...
```

👉 Esto rompe rutas en producción

---

## ✅ Lo correcto

### 1. Generar build

```bash
npm run build
```

Esto genera:

```
dist/
  index.html
  assets/
  js/
  css/
```

---

### 2. Subir el CONTENIDO de `dist/`

Ejemplo correcto en S3:

```
index.html
assets/
js/
css/
```

👉 `index.html` debe estar en la raíz del bucket

---

## 🧠 Regla clave

> S3 debe verse como si fuera el root de tu sitio web

---

# ⚙️ 2. Configuración de GitHub Actions

## ✅ Objetivo del pipeline

- Build automático
    
- Subida a S3
    
- Manejo correcto de cache
    
- Invalidación de CloudFront
    

---

## 🔥 YAML recomendado (producción)

```yaml
name: Deploy to S3 + CloudFront

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Create .env file
        run: echo "${{ secrets.ENV }}" > .env

      - name: Install dependencies
        run: npm ci

      - name: Generate build info
        run: |
          echo "VITE_BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> .env
          echo "VITE_BUILD_COMMIT=$(git rev-parse --short HEAD)" >> .env
          echo "VITE_BUILD_VERSION=$(node -p "require('./package.json').version")" >> .env

      - name: Build
        run: npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      # HTML sin cache
      - name: Upload index.html
        run: |
          aws s3 cp dist/index.html s3://TU_BUCKET/index.html \
            --cache-control "no-cache"

      # Assets con cache fuerte
      - name: Upload assets
        run: |
          aws s3 sync dist/assets/ s3://TU_BUCKET/assets/ \
            --cache-control "public, max-age=31536000, immutable"

      # Otros archivos
      - name: Sync rest
        run: |
          aws s3 sync dist/ s3://TU_BUCKET \
            --delete \
            --exclude "index.html" \
            --exclude "assets/*"

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

---

# 🔐 3. Secrets necesarios en GitHub

Debes configurar:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
CLOUDFRONT_DISTRIBUTION_ID
ENV (opcional)
```

---

# 💰 4. Manejo de cache (IMPORTANTE)

|Archivo|Cache|
|---|---|
|index.html|no-cache|
|JS/CSS (assets)|1 año|
|otros|normal|

---

## 🧠 ¿Por qué?

- Evita bugs de versiones viejas
    
- Mejora rendimiento
    
- Reduce costos en AWS
    

---

# ☁️ 5. Flujo final

```
GitHub → Build → S3 → CloudFront → Usuario
```

---

# 🚀 6. Requisitos en CloudFront

Configura:

- Default root object:
    

```
index.html
```

---

## Para SPA (Vue / React)

Agregar:

|Error|Acción|
|---|---|
|403|/index.html|
|404|/index.html|

---

# ⚠️ Errores comunes

❌ Subir `.zip`  
❌ Subir `dist/` como carpeta  
❌ No invalidar CloudFront  
❌ No configurar cache  
❌ Hacer bucket público

---

# 🧠 Resumen

✔️ Subir contenido de `dist/`  
✔️ Automatizar con GitHub Actions  
✔️ Usar CloudFront  
✔️ Configurar cache correctamente

---

# 🔥 Resultado

- 🚀 Deploy automático
    
- ⚡ Alto rendimiento
    
- 🔐 Seguro
    
- 💰 Optimizado en costos
    

---

---

Si quieres, el siguiente paso lógico sería:

👉 hacer un `.md` solo de **CloudFront bien configurado (OAC + SPA + dominio + HTTPS)** — ahí es donde más errores suelen pasar.