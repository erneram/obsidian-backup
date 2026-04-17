Muy buena pregunta — aquí está exactamente el comportamiento real de S3 + `sync`, que es clave entender 🔥

---

# 🧠 ¿Qué pasa cuando vuelves a hacer deploy?

Tú estás usando:

```bash
aws s3 sync dist/ s3://tu-bucket --delete
```

👉 Esto hace 3 cosas automáticamente:

---

## 🔄 1. Archivos modificados → se ACTUALIZAN

Si cambias algo en tu código:

```id="v3xk2p"
main.js
```

👉 En el siguiente deploy:

- S3 detecta que cambió
    
- Lo reemplaza
    

✔️ Automático  
✔️ No tienes que borrar nada manual

---

## ➕ 2. Archivos nuevos → se SUBEN

Si agregas algo nuevo:

```id="z8p1wq"
nuevo.js
```

👉 Se sube automáticamente al bucket

---

## ❌ 3. Archivos eliminados → se BORRAN (por `--delete`)

Si quitas algo de tu proyecto:

```id="n5t8ls"
old.js (ya no existe en dist)
```

👉 En el deploy:

- S3 lo elimina del bucket
    

---

# 🔥 Esto es EXACTAMENTE lo que quieres

👉 Mantiene S3 sincronizado con tu build

---

# ⚠️ IMPORTANTE (muy importante)

Si NO usas `--delete`:

👉 Pasaría esto:

- Archivos viejos se quedan en S3 ❌
    
- Basura acumulada ❌
    
- Bugs raros (JS antiguos cargándose) ❌
    

---

# 🧠 Ejemplo real completo

## Build 1:

```id="z4b7xp"
index.html
main.abc123.js
```

## Build 2:

```id="r9q2lm"
index.html
main.xyz789.js
```

👉 Con `--delete`:

- `main.abc123.js` ❌ eliminado
    
- `main.xyz789.js` ✅ agregado
    

---

# 💡 ¿Y CloudFront?

👉 Aquí entra otro detalle importante:

Aunque S3 ya tenga los cambios…

👉 CloudFront puede seguir sirviendo lo viejo 😬

---

## 🔄 Por eso haces esto:

```bash
aws cloudfront create-invalidation --paths "/*"
```

👉 Le dices:

> “Olvida el cache y trae lo nuevo”

---

# 🧠 Flujo completo

```id="z2m9ks"
Build → S3 sync → elimina/actualiza → CloudFront invalidation → usuarios ven lo nuevo
```

---

# ⚠️ Edge case importante

👉 Si NO invalidas CloudFront:

- usuarios ven versión vieja ❌
    
- aunque S3 esté actualizado
    

---

# 🧩 Conclusión clara

|Acción|Resultado|
|---|---|
|Modificas archivo|se actualiza|
|Agregas archivo|se sube|
|Eliminas archivo|se borra (gracias a `--delete`)|

---

# 🚀 Resumen en una línea

👉 **S3 siempre queda idéntico a tu carpeta `dist/`**

---

Si quieres, el siguiente nivel (muy útil) es:

👉 cómo evitar invalidar todo CloudFront (para ahorrar dinero y hacerlo más eficiente)