---
title: Tenancy – Valet + /etc/hosts
tags: [laravel, tenancy, valet, hosts, dns, local]
---

# 🌐 Valet + /etc/hosts con Tenancy

Cómo funciona el routing local para multi-tenancy: Valet, subdominios y el archivo `/etc/hosts`.

---

## ¿Cómo identifica Tenancy al tenant?

Por defecto, via **dominio**. El middleware `InitializeTenancyByDomain` extrae el host del request y busca en la tabla `domains` qué tenant corresponde.

```txt
Request a foo.localhost
  → middleware busca domain = "foo.localhost"
    → encuentra tenant "acme"
      → inicializa tenancy con ese tenant
```

---

## Valet y cómo funciona

Valet sirve aplicaciones Laravel localmente bajo dominios `.test` (y también `.localhost`).

```bash
# Dentro del directorio del proyecto
valet link myapp
# → disponible en http://myapp.test
```

Valet resuelve `*.test` y `*.localhost` automáticamente via su propio DNS daemon (dnsmasq), sin necesidad de editar `/etc/hosts` para el dominio principal.

### Dominio central (la app principal)

```php
// config/tenancy.php
'central_domains' => [
    'myapp.test',
],
```

Este dominio es el de la app central (admin, login, registro). No inicializa tenancy.

---

## /etc/hosts — qué es y qué tiene adentro

`/etc/hosts` es un archivo del sistema operativo que mapea nombres de dominio a IPs **antes de consultar DNS**. Es la forma más rápida de resolver un dominio localmente.

```bash
# Ver contenido
cat /etc/hosts
```

Estructura típica:

```
##
# Host Database
#
# localhost is used to configure the loopback interface
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost
```

---

## Cuándo necesitas editar /etc/hosts

**Con Valet**: normalmente NO necesitas editar `/etc/hosts` para subdominios `.test` o `.localhost` porque dnsmasq los resuelve automáticamente como wildcard (`*.test → 127.0.0.1`).

**Sin Valet** (o con subdominios que no resuelven): sí necesitas agregar entradas manualmente.

### Agregar subdominios de tenants

```bash
sudo nano /etc/hosts
```

Agregar:

```
127.0.0.1   myapp.test
127.0.0.1   foo.localhost
127.0.0.1   bar.localhost
127.0.0.1   acme.localhost
```

Cada línea le dice al OS: "cuando alguien pida `foo.localhost`, manda el request a `127.0.0.1` (tu máquina local)".

---

## Flujo completo local con Valet

```txt
Browser pide: http://foo.localhost

1. OS consulta /etc/hosts → no tiene entrada explícita
2. Valet dnsmasq resuelve *.localhost → 127.0.0.1
3. Request llega a Valet (nginx local)
4. Valet lo enruta a tu app Laravel
5. Middleware InitializeTenancyByDomain detecta "foo.localhost"
6. Busca en tabla `domains` → encuentra tenant "foo"
7. Cambia conexión DB al tenant "foo"
8. Request procesa en contexto de ese tenant
```

---

## Crear tenants con dominio local

```php
$tenant = App\Models\Tenant::create(['id' => 'foo']);
$tenant->domains()->create(['domain' => 'foo.localhost']);

$tenant2 = App\Models\Tenant::create(['id' => 'bar']);
$tenant2->domains()->create(['domain' => 'bar.localhost']);
```

Acceder en el browser:
- `http://foo.localhost` → tenant "foo"
- `http://bar.localhost` → tenant "bar"
- `http://myapp.test` → app central

---

## Verificar que Valet resuelve el subdominio

```bash
ping foo.localhost
# Debe responder desde 127.0.0.1
```

Si no responde, agregar la entrada en `/etc/hosts` manualmente.

---

## config/tenancy.php — configuración clave

```php
return [
    'tenant_model' => App\Models\Tenant::class,

    // Dominio(s) de la app central (NO inicializan tenancy)
    'central_domains' => [
        'myapp.test',
        'localhost', // solo si usas localhost puro
    ],

    // Cómo se identifica el tenant
    // InitializeTenancyByDomain usa el host del request
];
```

---

## ↩️ Navegación

- [[Backend/Laravel/Packages/Tenancy/Tenancy|🏢 Tenancy Hub]]
