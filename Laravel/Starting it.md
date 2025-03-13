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

# **Accede a tu Proyecto Laravel**
***
```sh
cd ~/Projects/inventory-system
```
###### (Opcional) Limpia la caché de Laravel si has realizado cambios o para asegurarte de que se carguen las configuraciones

```sh
php artisan config:clear 
php artisan cache:clear 
php artisan route:clear 
php artisan view:clear  
```
###### Abre tu aplicación en el navegador. Por ejemplo, si usas el nombre inventory-system
```  
http://inventory-system.test
```
  
## **Verifica que todo funcione correctamente**
***
###### Si ves tu aplicación o la página de bienvenida de Laravel, significa que se ha levantado correctamente.
###### Si necesitas trabajar con la API o las rutas web, asegúrate de que estén definidas en tus archivos de rutas (routes/web.php o routes/api.php).

# ERROR EN VENDOR
***
```sh
rm -rf vendor
composer install
```
