

## CREACION DEL SERVIDOR

## Certificado SSH
1. En la sección de "Network & Security" seleccionar la opción de "Key Pairs".
2. Crear una nueva llave seleccionando el tipo RSA y el formato privado .pem
3. Descargar el archivo y guardarlo en un lugar seguro.

## Instancia EC2
1. En el dashboard principal de AWS navegar hacia el servicio de EC2.
2. Buscar la opción de "Launch instance".
3. Nombrar el recurso.
4. Seleccionar el sistema operativo (Ubuntu).
5. Seleccionar el tipo de instancia (t2.micro o el tamaño deseado).
6. Seleccionar el key pair generado en el paso anterior.
7. Validar que esté seleccionada la opción de "Allow SSH traffic from" y que el valor sea "Anywhere"
8. Seleccionar la opción de "Allow HTTPS traffic from the internet"
9. Seleccionar la opción de "Allow HTTP traffic from the internet".
10. Definir 30GB para el disco.
11. Click en "Launch instance"
12. Verificar que el estado de la instancia sea "Running" si no toca esperar.

## IP Estática

Las IP estáticas se llaman "Elastic IPs" y nos permiten asociar una dirección que no cambie a nuestra nueva instancia de EC2.

Para crear la IP que tenemos que navegar a "Elastic IPs" en la sección de "Network & Security" y seleccionar la opción de "Allocate Elastic IP Address"

Cuando tengamos alocada la dirección estática podemos procesder a asociarla a la instancia que creamos en el paso anterior. Esto se puede hacer seleccionando la IP y usando la opción de "Associate Elastic IP Address" del menú de Actions.

## Conectarse por SSH

Si seleccionamos la instancia vamos a poder ver un menú que dice "Connect" si vamos a la pestaña que dice "SSH Client" vamos a ver instrucciones para conectarnos.

Es importante que la llave que descargamos en el paso 1 tenga permisos limitados por lo que podemos correr:

```
chmod 400 "NOMBRE_LLAVE.pem"
```

Si usamos la misma configuración que esta guía podemos conectarnos con el siguiente comando:

```
ssh -i NOMBRE_LLAVE.pem ubuntu@IP_PUBLICA
```

## CONFIGURACIONES Y MEJORES PRÁCTICAS

## Cambiar la clave del root

Es de suma seguridad poder cambiar la clave del root. En el caso de AWS esta clave no existe ya que el servidor no es accesible por medio de login con clave sino que solo por certificado.

Con el siguiente comando podemos cambiar la clave del root:

```
sudo passwd root
```

Introducimos una clave _apropiada_ y la confirmamos.

Luego cambiamos la clave del usuario default:

```
sudo passwd ubuntu
```

Introducimos una clave _apropiada_ y la confirmamos.

## Mejorar la configuración de SSH

El archivo de configuración para el SSH lo podemos encontrar en:

```
/etc/ssh/sshd_config
```

En este archivo está la configración del servicio. Tomemos en cuenta que al cambiar esta configuración es necesario reiniciar el servidor.

Cambiar la configuración a esto:

```
PasswordAuthentication yes
PubkeyAuthentication no
UsePAM yes
PermitRootLogin yes (SOLO SI ES NECESARIO!)
```

Ya que se han producido cambios en la configuración del servicio se debe reiniciar para que esos cambios tomen efecto, para hacer eso corremos:

```
service ssh restart
```

Ahora debemos indicarle al AWS que vamos a conectarnos usando clave (lo que a ellos no les gusta).

```
sudo vi /etc/cloud/cloud.cfg
```
Agregamos la siguiente línea:

```
ssh_pwauth: true
```

y reiniciamos:

```
sudo cloud-init clean
sudo cloud-init init
```

Para conectarnos a nuestra nueva instancia de EC2 debemos utilizar el comando SSH desde nuestra terminal o cliente se SSH (PuTTy, Terminus, Terminal, etc.)

```
ssh <user>@<ip>
```

Por ejemplo:

```
ssh root@127.0.0.1
```

Para mejorar la seguridad es recomendado cambiar el puerto de SSH para que no sea el default 22. Para hacer esto nuevamente modificamos el archivo

```
vi /etc/ssh/sshd_config
```

pero ahora buscamos la llave "Port" y asignamos el nuevo valor

```
Port 30
```

y nuevamente reiniciamos el servicio:

```
service ssh restart
```

Finalmente en la sección de configuración del AWS buscamos la pestaña de seguridad buscamos el grupo de seguridad y hacemos click sobre el nombre. Aquí debemos agregar el puerto que definimos en la configuración para que podamos conectarnos usando el puerto que definimos. Es una nueva "Inbound rule"

Ahora ya podemos conectarnos definiendo el puerto y el flag -p

```
ssh <user>@<ip> -p <puerto>
```

Por ejemplo:

```
ssh root@127.0.0.1 -p 30
```

NOTA: Si ya se configuro el puerto, comentar el puerto default.
```
#Port 22
Port 30
```
Luego reiniciar la instancia en el servidor para poder acceder.
## Instalar Nginx

Antes de instalar paquetes, primero es recomendado actualizar, para hacer eso podemos usar el comando:

```
apt update
```

Al tener los repocitorios actualizados podemos instalar Nginx corriendo:

```
apt install nginx -y
```

Luego de instalarlo debemos levantar el servicio corriendo:

```
service nginx start
```

para verificar el estado del servicio podemos utilizar:

```
service nginx status
```

Para verificar que funcionó la instalación podemos navegar a la IP de la instancia y debemos ver un mensaje de Nginx.

El root del web server está en:

```
/var/www/html
```

### Permitir ver directorios

Para que nos permita ver directorios el Nginx es necesario modificar el archivo de configuración del servicio:

```
vi /etc/nginx/nginx.conf
```

y dentro de la llave de http agregamos lo siguiente:

```
autoindex on;
```

Y como cada vez que hacemos cambios a un archivo de config, debemos reiniciar el servicio:

```
service nginx restart
```
