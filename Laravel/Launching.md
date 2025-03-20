# **Levantar Laravel Valet**
***
**Inicia MySQL con Brew (si lo usas)**
###### Abre la terminal y ejecuta:bash  
```sh
brew services start mysq
```
###### Verifica que MySQL esté corriendo:bash  
```sh
brew services list
```
###### Si MySQL ya está corriendo, no es necesario reiniciarlo; de lo contrario, usa:bash  
```sh 
brew services restart mysql
```
  
**Inicia Laravel Valet**
***
###### Reinicia Valet para asegurarte de que todos los servicios estén activos:bash  
```sh  
valet restart  
```
###### Si aún no tienes tu carpeta de proyectos "parqueada", navega al directorio donde están tus proyectos y ejecuta:bash  
```sh
cd ~/Projects  # O el directorio que uses para tus proyectos
valet park
```
###### Esto hará que cualquier carpeta dentro de ese directorio se asocie a un dominio .test.

#####  (opcional) Si es la primera vez que ejecutas Valet en tu sistema o tienes problemas con la configuración, ejecuta:
```sh 
valet install
```

# **Accede a tu Proyecto Laravel**

---
###### Navega a la carpeta de tu proyecto Laravel:
```sh
cd ~/Projects/inventory-system
```
###### Laravel necesita un archivo `.env`. Si no existe, crea uno desde el ejemplo:
```sh
cp .env.example .env
```
###### REVISAR VALORES:
```sh
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=inventory_system
DB_USERNAME=tu_usuario
DB_PASSWORD=tu_contraseña

SESSION_DRIVER=file # NOT database
```
###### para luego:
```sh
 php artisan config:clear
```

###### Si es la primera vez que descargas el proyecto desde Git, asegúrate de instalar las dependencias de Composer:
```sh 
composer install
```

######  Si necesitas instalar las dependencias de frontend (para Vite, Tailwind, etc.), ejecuta:

```sh
yarn / npm install
```
###### (Opcional) Limpia la caché de Laravel si has realizado cambios o para asegurarte de que se carguen las configuraciones

```sh
php artisan config:clear 
php artisan cache:clear 
php artisan route:clear 
php artisan view:clear  
```
###### Si Laravel sigue sin funcionar correctamente después de limpiar la caché, puedes intentar regenerar la clave de la aplicación:
```sh
php artisan key:generate
```


###### Abre tu aplicación en el navegador. Por ejemplo, si usas el nombre inventory-system

```
http://inventory-system.test
```

## **Verifica que todo funcione correctamente**

---

###### Si ves tu aplicación o la página de bienvenida de Laravel, significa que se ha levantado correctamente.

###### Si necesitas trabajar con la API o las rutas web, asegúrate de que estén definidas en tus archivos de rutas (routes/web.php o routes/api.php).

# ERROR EN VENDOR

---
###### Si ves un error relacionado con `vendor/autoload.php`, ejecuta:
```sh
rm -rf vendor
composer install
```
###### Si persiste, intenta regenerar el autoload de Composer:
```sh
composer dump-autoload
```

###### Si Laravel no encuentra `manifest.json`, asegúrate de correr:
```sh
npm run build php artisan optimize:clear
```
