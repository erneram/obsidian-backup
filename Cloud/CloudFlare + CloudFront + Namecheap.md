Aquí tienes un `.md` enfocado específicamente en **conectar tu dominio (Cloudflare) con CloudFront**, continuando desde la guía anterior:

---

# 🌐 Guía: Conectar dominio (Cloudflare) con CloudFront + S3

Esta guía explica cómo conectar tu dominio (ej: `hospitalbelen.com`) usando:

- 🌍 Cloudflare (DNS)
    
- ☁️ CloudFront (CDN)
    
- 🪣 S3 (origen)
    

---

# 🧠 Arquitectura final

```txt
Usuario → Cloudflare → CloudFront → S3
```

---

# 🔍 1. Obtener dominio de CloudFront

Ve a:

👉 AWS → CloudFront → Distributions

Copia el campo:

```txt
Domain name: d3k2abc123xyz.cloudfront.net
```

⚠️ Este valor es único  
❌ NO es `dxxxxx` literal  
✔️ Debes usar tu valor real

---

# 🌐 2. Configurar DNS en Cloudflare

Ve a:

👉 Cloudflare → tu dominio → DNS

---

## 🔹 A. Root domain (@)

Esto representa:

```txt
hospitalbelen.com
```

Configura:

|Campo|Valor|
|---|---|
|Type|CNAME|
|Name|@|
|Target|d3k2abc123xyz.cloudfront.net|
|Proxy|🟠 ON|
|TTL|Auto|

---

## 🔹 B. Subdominio WWW

Esto representa:

```txt
www.hospitalbelen.com
```

Configura:

|Campo|Valor|
|---|---|
|Type|CNAME|
|Name|www|
|Target|d3k2abc123xyz.cloudfront.net|
|Proxy|🟠 ON|
|TTL|Auto|

---

# 🧠 Regla clave

👉 Ambos deben apuntar al mismo target:

```txt
d3k2abc123xyz.cloudfront.net
```

---

# 🔥 ¿Por qué el mismo target?

Porque:

- CloudFront es el CDN central
    
- S3 es el origen interno
    
- Cloudflare solo redirige tráfico
    

---

# ⚠️ Errores comunes

## ❌ Usar "dxxxxx" literal

👉 Es solo ejemplo de AWS

---

## ❌ No usar @

👉 El dominio raíz no funcionará

---

## ❌ Proxy en gris (DNS only)

👉 Pierdes:

- caché
    
- seguridad
    
- WAF
    

✔️ Debe estar:

```txt
🟠 Proxy ON
```

---

## ❌ Targets diferentes

👉 Ambos deben apuntar al mismo CloudFront

---

# 🔐 3. Configurar SSL en Cloudflare

Ve a:

👉 SSL/TLS → Overview

Selecciona:

```txt
Full (strict)
```

---

# 🚀 4. Resultado final

Después de configurar:

```txt
https://hospitalbelen.com
https://www.hospitalbelen.com
```

Ambos funcionarán correctamente 🚀

---

# 🔁 Flujo completo

```txt
Usuario → hospitalbelen.com → Cloudflare → CloudFront → S3
```

---

# 💡 Opcional (mejora)

Puedes decidir:

- redirigir `www → root`
    
- o `root → www`
    

(esto se configura después)

---

# 🧠 Resumen

✔️ Usar dominio real de CloudFront  
✔️ Crear 2 CNAME (root y www)  
✔️ Ambos con mismo target  
✔️ Proxy activado  
✔️ SSL en Full (strict)

---

# 🔥 Estado esperado

- CDN funcionando ⚡
    
- HTTPS activo 🔐
    
- Bucket privado protegido 🔒
    
- Deploy automático listo 🚀
    

---

---

Si quieres, el siguiente `.md` te lo puedo hacer de:

👉 **redirecciones (www ↔ root) + headers + optimización avanzada de CloudFront** (nivel producción real).