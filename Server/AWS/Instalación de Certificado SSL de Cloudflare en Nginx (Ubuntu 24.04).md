## 1. Generar el Certificado SSL en Cloudflare

1. Accede al **panel de Cloudflare**.
2. Selecciona tu dominio y ve a **SSL/TLS** → **Origen del Servidor**.
3. Haz clic en **Crear Certificado**.
4. Elige la opción **Permitir que Cloudflare genere la clave privada y CSR**.
5. Selecciona **15 años** como el período de expiración.
6. Agrega tu dominio y subdominios (ej. `tudominio.com`, `www.tudominio.com`).
7. Haz clic en **Crear**.
8. Copia y guarda:
   - **Certificado de Origen** (clave pública).
   - **Clave Privada**.

---
## 2. Guardar el Certificado en el Servidor

1. **Conéctate a tu instancia de AWS EC2**:
   ```bash
   ssh usuario@tu-servidor
   ```

2. **Crea el directorio para los certificados**:
   ```bash
   sudo mkdir -p /etc/nginx/ssl
   ```

3. **Guarda el certificado y la clave privada**:
   - Abre un archivo para el **certificado**:
     ```bash
     sudo nano /etc/ssl/cloudflare.crt
     ```
     - Pega el **Certificado de Origen** de Cloudflare.
     - Guarda y cierra (`CTRL + X`, luego `Y` y `Enter`).

   - Abre un archivo para la **clave privada**:
     ```bash
     sudo nano /etc/ssl/cloudflare.key
     ```
     - Pega la **Clave Privada** de Cloudflare.
     - Guarda y cierra.
	NOTA: SE DEBE DE COPIAR TODO! INCLUSO EL TEXTO DE INICIO Y FINAL
	
4. **Asegura los permisos correctos**:
   ```bash
   sudo chmod 600 /etc/ssl/cloudflare.*
   ```

---

## 3. Configurar Nginx para Usar SSL (Virtual Host)

Un Virtual Host es una configuración en un servidor web que permite que un mismo servidor físico o instancia de servidor maneje múltiples sitios web o aplicaciones de forma independiente.

En términos prácticos, los Virtual Hosts permiten que un servidor pueda servir diferentes dominios o subdominios desde una misma máquina con configuraciones separadas.

1. **Edita o crea un archivo de configuración para tu dominio**:
   ```bash
   sudo nano /etc/nginx/sites-available/tudominio.com.conf
   ```

2. **Añade la siguiente configuración**:

   ```nginx
   server {
       listen 80;
       server_name tudominio.com www.tudominio.com;
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       server_name tudominio.com www.tudominio.com;

       ssl_certificate /etc/ssl/cloudflare.crt;
       ssl_certificate_key /etc/ssl/cloudflare.key;

       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_ciphers HIGH:!aNULL:!MD5;

       location / {
           root /var/www/html;
           index index.html index.htm;
       }
   }
   ```

3. **Habilita el sitio**:
   ```bash
   sudo ln -s /etc/nginx/sites-available/tudominio.com.conf /etc/nginx/sites-enabled/
   ```

---

## 4. Reiniciar Nginx y Aplicar los Cambios

1. **Verifica que la configuración de Nginx sea válida**:
   ```bash
   sudo nginx -t
   ```

2. **Reinicia Nginx**:
   ```bash
   sudo systemctl restart nginx
   ```

---

## 5. Configurar Cloudflare para Usar SSL

1. En el **panel de Cloudflare**, ve a **SSL/TLS** → **Resumen**.
2. Cambia el modo SSL a **"Completo (estricto)"**.

---

## 6. Verificar que el Certificado SSL Funciona

1. Accede a tu sitio en el navegador:
   ```
   https://tudominio.com
   ```
2. Usa **SSL Labs** para verificar el certificado:
   ```
   https://www.ssllabs.com/ssltest/analyze.html?d=tudominio.com
   ```

Ahora tu servidor **AWS EC2** con Nginx está protegido con un **Certificado SSL de Cloudflare**. 🚀


---

## ↩️ Navegación
- [[Server/AWS/AWS|☁️ AWS]] → [[Server/Server|🖥️ Server]] → [[HOME|🏠 HOME]]
