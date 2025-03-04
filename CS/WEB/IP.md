IP identificador personal

Octeto 0 - 255 en cada octeto “xxx.xxx.x.x” (Poca escalabilidad a largo plazo; Quedarse sin direcciones IP)

IPv4 Cuatro segmentos de números separados por puntos

IPv6 separación por 2 puntos y mas largas las direcciones

Diseñado para soportar un número mucho mayor de dispositivos

Redes internas

Pueden existir mismas IP pero en diferentes redes.

Redes externas

Puerto: Indicador del servicio que estoy corriendo (0 - 65535)

0 - 1023: Puertos conocidos 80-HTTP, 23-telnet, 443-HTTPS

1024 - 49151: Puerto registrados para aplicaciones (uso general)

49152 - 65535: Puertos dinámicos o privados (puertos de nadie)

Concurrencia: Multiples personas conectadas a la vez consumiendo

Para poner pagina en internet se necesita

1. Computadora con IP publica

HTTP

Escucha el servidor HTTP las solicitudes del puerot 80 y devuelve

GET: Solicita uin recurso del servidor como una pagina web

POST: Envia datos al servidor, guardar información

DNS:

Domain Name System, traduce nombres de dominio legibles por humanos “[example.com](http://example.com)”

Facilita la utilización de internet (no recordar la dirección ip)

Gestionar dominios y subdominios

Utilizan “Zones tables” donde estan los listados de dominios e ip (se compra el derecho de ponerse en esa tabla) Pagar dominio significa es resevar un espacio para ponerlo en el espacio de zonas existe una ip xx.x.x.x.x ⇒ Lo que se pago

Configurar un dominio (en goDaddy)

IPv4: A

IPv6: AAAA

Alias para otro dominio: CNAME

Servidores de correo: MX

Información adicional: TXT

lan : Local Area Network