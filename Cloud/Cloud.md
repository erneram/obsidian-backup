---
title: Cloud Hub
tags: [hub, cloud, aws, cloudfront, s3, cloudflare]
---

# ☁️ Cloud – Hub

Documentación del stack de deployment en la nube: **S3 + CloudFront + Cloudflare**.

---

## 🧠 Arquitectura general

```txt
Usuario → Cloudflare (DNS) → CloudFront (CDN) → S3 (origen)
```

El bucket S3 es privado. Nadie accede directamente. CloudFront sirve los archivos, Cloudflare maneja DNS y SSL.

---

## 📂 Páginas

### 🪣 Infraestructura

- [[Cloud/Bucket|Bucket]] — crear el bucket S3 con configuración correcta (privado, cifrado, sin acceso público).
- [[Cloud/CloudFlare + CloudFront + Namecheap|CloudFlare + CloudFront + Namecheap]] — conectar dominio en Cloudflare con una distribución CloudFront (CNAME, Proxy ON, SSL Full Strict).

### 🚀 Deploy

- [[Cloud/Deploy.yml and dist|Deploy.yml and dist]] — pipeline de GitHub Actions: build → S3 sync → invalidar CloudFront. Manejo de cache por tipo de archivo.
- [[Cloud/Underhood dist|Underhood dist]] — cómo funciona `aws s3 sync --delete` por dentro: archivos modificados, nuevos y eliminados. Por qué invalidar CloudFront.

---

## 🔄 Flujo completo de deploy

```txt
git push main
  → GitHub Actions
    → npm run build
    → aws s3 cp (index.html sin cache)
    → aws s3 sync (assets con cache 1 año)
    → aws s3 sync --delete (resto)
    → aws cloudfront create-invalidation
  → Usuarios ven la nueva versión
```

---

## ⚠️ Reglas clave

| Regla | Por qué |
|-------|---------|
| `index.html` con `no-cache` | Siempre carga la versión más reciente |
| Assets con `max-age=31536000, immutable` | El hash en el nombre garantiza unicidad |
| `--delete` en sync | Mantiene S3 limpio, sin basura acumulada |
| Invalidar CloudFront | Sin esto los usuarios siguen viendo lo viejo |
| Proxy Cloudflare ON | Activa caché, WAF y seguridad |

---

## ↩️ Navegación

- [[HOME|🏠 HOME]]
