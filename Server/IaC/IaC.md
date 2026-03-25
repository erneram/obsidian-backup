---
title: IaC & Contenedores – Hub
tags: [hub, infrastructure, iac, docker, kubernetes, terraform]
---

# 🏗️ IaC & Contenedores – Hub

Índice del área **Infrastructure as Code (IaC) y Contenedores**. Cubre las herramientas estándar para definir, empaquetar y orquestar infraestructura de forma reproducible y automatizada.

---

## 🗂️ Contenido

- [[Server/IaC/Docker|Docker]] — empaqueta aplicaciones en contenedores portables y aislados.
- [[Server/IaC/Kubernetes|Kubernetes]] — orquesta y escala contenedores en producción.
- [[Server/IaC/Terraform|Terraform]] — define y provisiona infraestructura en la nube con código.

---

## 🧠 Cuándo usar cada uno

| Herramienta | Resuelve |
|-------------|----------|
| **Docker** | Empaquetar la app + sus dependencias para que corra igual en cualquier entorno |
| **Kubernetes** | Escalar, distribuir y mantener vivos contenedores en múltiples servidores |
| **Terraform** | Crear/destruir recursos cloud (EC2, RDS, VPCs…) desde código versionable |

> En proyectos grandes se usan los tres juntos: **Terraform** crea la infraestructura, **Docker** empaqueta la app, y **Kubernetes** la ejecuta.

---

## 🔗 Relacionado con otras áreas

- [[Server/Server|Server Hub]] — índice general de infraestructura.
- [[Server/AWS/AWS|AWS Hub]] — servicios cloud donde corre la infraestructura.
- [[Projects/SchoolAid/SchoolAid|SchoolAid]] — proyecto desplegado en la nube.

---

## ↩️ Navegación
- [[Server/Server|🖥️ Server]] → [[HOME|🏠 HOME]]
