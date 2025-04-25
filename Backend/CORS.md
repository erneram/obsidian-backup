Cors es una politica de seguridad utilizada especificamente en navegadores utilizadas para controlar solicitudes HTTP entre diferentes dominios. 
Permite proteger a usuarios de ataques Cross-Site Scripting (XSS)

Para ello es necesario especificar los origenes los cuales van a tener permisos de tocar los recursos 

Una request debe de contener lo siguiente:
- Metodo http permitido
	- [`GET`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/GET)
	- [`HEAD`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/HEAD)
	- [`POST`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/POST)
- Headers permitidos
	- - [`Accept`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Accept)
	- [`Accept-Language`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Accept-Language)
	- [`Content-Language`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Language)
	- [`Content-Type`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Type)
	- [`Range`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Range)
- Content Type
	- `application/x-www-form-urlencoded`
	- `multipart/form-data`
	- `text/plain`

# Preflighted requests
---
Sucede cuando primero se envia al servidor un HTTP request utilizando el metodo `OPTIONS` para determinar si el request actual es seguro de enviar. Estas request tienen que ser prevalidadas para que no tengan implicaciones al usuario

# The HTTP response headers
---
1. Access-Control-Allow-Origin
	Sirve para especificar el origen de donde debe de venir la solicitud para permitirle el acceso. Utilizando `*` estamos diciendo que se aceptan solicitudes de cualquier lugar.
2. Access-Control-Expose-Headers

3. Access-Control-Max-Age
	Especifica el tiempo exacto en el que los resultados de la solicitud se puedan guardar en cache, se manda por `<delta_seconds>` y siempre depende del navegador cual es el máximo.
4. Access-Control-Allow-Credentials
	Se utiliza generalmente como una solicitud previa, al devolver un `true` lo que indica que ya en la solicitud real, ya se puede realizar con credenciales. En caso contrario no.
	Al hacer un metodo `GET` no se hace una solicitud previa.
5. Access-Control-Allow-Methods
	Este devuelve un listado de los métodos HTTP que son permitidos hacer en la ruta que se enviaron.
6. Access-Control-Allow-Headers
	Se utiliza en la respuesta a una solicitud Preflighted para indicar que encabezados HTTP se pueden utilizar