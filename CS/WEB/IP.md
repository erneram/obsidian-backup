---
title: Redes – IP, Puertos, HTTP y DNS
tags: [networking, ip, http, dns]
---

> [!summary] En 10 segundos
> - **IP** identifica un dispositivo en una red. **IPv4** (32 bits) y **IPv6** (128 bits).
> - **Puertos** identifican servicios en el host (0–65535).
> - **HTTP** usa puertos 80/443 y métodos como **GET**/**POST**.
> - **DNS** traduce nombres a IP y se configura con **registros** (A, AAAA, CNAME, MX, TXT).

---

# 1) Dirección IP

## 1.1 IPv4
- Formato: `xxx.xxx.xxx.xxx` (4 octetos, **0–255** cada uno).
- Ejemplo válido: `192.168.1.10`
- Capacidad limitada ⇒ **poca escalabilidad** a muy largo plazo.

## 1.2 IPv6
- Formato: 8 grupos hex separados por `:`  
  Ej.: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Se puede abreviar ceros: `2001:db8:85a3::8a2e:370:7334`
- **Diseñado para soportar muchísimos más dispositivos.**

> [!tip] Redes internas vs externas
> - **Red interna (LAN):** pueden existir **misma IP** en LANs diferentes (p. ej., `192.168.0.10` en dos casas distintas).
> - **Red externa (Internet):** IP **pública** única hacia Internet.

---

# 2) Puertos

Los puertos identifican **servicios** en un mismo host. Rango **0–65535**.

| Rango            | Nombre                               | Uso típico                               |
|------------------|--------------------------------------|------------------------------------------|
| 0–1023           | Puertos **conocidos** (well-known)   | 80=HTTP, 443=HTTPS, 23=Telnet, 22=SSH    |
| 1024–49151       | Puertos **registrados**              | Apps/servicios de uso general            |
| 49152–65535      | **Dinámicos/privados** (ephemeral)   | Conexiones temporales del cliente        |

> [!note] Concurrencia
> Varias personas conectadas **al mismo tiempo** consumiendo un servicio (múltiples sockets/IP:puerto).

---

# 3) Poner una página en Internet (mínimos)

1) **Computadora/Servidor con IP pública** (o un VPS).  
2) **Servicio HTTP/HTTPS** escuchando en **80/443**.  
3) **Dominio** apuntando (DNS) a la IP pública.  
4) (Recomendado) **Certificado TLS** (HTTPS – puerto 443).

---

# 4) HTTP (capa aplicación)

- El servidor **escucha** en **80 (HTTP)** o **443 (HTTPS)**.
- Métodos comunes:
  - **GET** → Solicita un recurso (página, imagen, API read).
  - **POST** → Envía datos al servidor (formularios, API write).

```http
GET /index.html HTTP/1.1
Host: ejemplo.com

POST /api/contact HTTP/1.1
Host: ejemplo.com
Content-Type: application/json

{"name":"Sam","msg":"Hola"}
