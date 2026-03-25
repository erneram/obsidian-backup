---
title: Terraform
tags: [infrastructure, terraform, iac, cloud, devops]
---

# 🌍 Terraform

---

## ¿Qué es?

Terraform es una herramienta de **Infrastructure as Code (IaC)** creada por HashiCorp. Permite definir infraestructura cloud (servidores, bases de datos, redes, balanceadores…) usando archivos de texto, y luego **crear, modificar o destruir** esa infraestructura de forma automática y reproducible.

En lugar de hacer clic en la consola de AWS o Azure para crear un EC2, escribes código y Terraform lo provisiona por ti.

---

## ¿Cómo funciona?

### Flujo de trabajo

```
Escribe .tf  ──terraform init──►  Descarga providers
             ──terraform plan──►  Muestra qué va a cambiar (dry-run)
             ──terraform apply──► Crea/modifica la infraestructura real
             ──terraform destroy─► Elimina todo lo que creó
```

### Componentes clave

**Provider** — el plugin que habla con cada plataforma (AWS, GCP, Azure, Cloudflare…).

**Resource** — el bloque base. Cada recurso representa una pieza de infraestructura (una instancia EC2, un bucket S3, un registro DNS…).

**State (`terraform.tfstate`)** — archivo donde Terraform guarda el estado actual de la infraestructura. Es la fuente de verdad que le permite saber qué existe y qué necesita cambiar.

**Module** — agrupación reutilizable de recursos. Equivale a una función: parametrizable y llamable desde múltiples lugares.

**Variable / Output** — las variables parametrizan los valores; los outputs exponen información (como la IP de un servidor creado) para usarla en otros módulos o mostrarla al final.

### Cómo calcula los cambios

Terraform compara el **estado actual** (en `tfstate`) con el **estado deseado** (en tus archivos `.tf`) y genera un plan con exactamente qué va a crear, modificar o eliminar.

```
Estado actual (tfstate) ──►┐
                            ├─► DIFF ──► Plan ──► Apply
Estado deseado (.tf)   ──►┘
```

---

## ¿Qué hace?

- **Provisiona** recursos cloud: EC2, RDS, VPC, S3, ECS, EKS, Route 53, etc.
- **Versiona** la infraestructura: los `.tf` van en git, igual que el código.
- **Detecta drift**: si alguien cambia algo manualmente en la consola, Terraform lo detecta.
- **Planifica antes de actuar**: el `terraform plan` muestra los cambios antes de aplicarlos.
- **Gestiona dependencias**: sabe en qué orden crear recursos (primero la VPC, luego la subred, luego el EC2).
- **Es multi-cloud**: el mismo flujo de trabajo para AWS, GCP, Azure, Cloudflare, etc.

---

## Configuración para un proyecto nuevo

### Estructura de carpetas recomendada

```
infra/
├── main.tf          # Recursos principales
├── variables.tf     # Declaración de variables
├── outputs.tf       # Valores que expones al final
├── providers.tf     # Configuración del provider
├── terraform.tfvars # Valores concretos de variables (no subir a git si tiene secrets)
└── modules/
    └── ec2/         # Módulo reutilizable para EC2
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### `providers.tf`

```hcl
terraform {
  required_version = ">= 1.7"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Backend remoto: guarda el tfstate en S3 (recomendado para equipos)
  backend "s3" {
    bucket         = "mi-proyecto-tfstate"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"   # evita que dos devs apliquen a la vez
  }
}

provider "aws" {
  region = var.aws_region
}
```

### `variables.tf`

```hcl
variable "aws_region" {
  description = "Región de AWS donde se despliega la infraestructura"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Ambiente: development, staging, production"
  type        = string
}

variable "instance_type" {
  description = "Tipo de instancia EC2"
  type        = string
  default     = "t3.micro"
}

variable "db_password" {
  description = "Contraseña de la base de datos"
  type        = string
  sensitive   = true    # Terraform no la imprime en los logs
}
```

### `main.tf` — ejemplo: EC2 + RDS + Security Groups

```hcl
# VPC y subred
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "${var.environment}-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "${var.aws_region}a"
}

# Security group: permite SSH y HTTP
resource "aws_security_group" "web" {
  name   = "${var.environment}-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Instancia EC2
resource "aws_instance" "web" {
  ami                    = "ami-0c02fb55956c7d316"  # Amazon Linux 2023
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name        = "${var.environment}-web"
    Environment = var.environment
  }
}

# Base de datos RDS MySQL
resource "aws_db_instance" "main" {
  identifier        = "${var.environment}-db"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  db_name           = "app_db"
  username          = "admin"
  password          = var.db_password
  skip_final_snapshot = true

  tags = { Environment = var.environment }
}
```

### `outputs.tf`

```hcl
output "web_public_ip" {
  description = "IP pública del servidor web"
  value       = aws_instance.web.public_ip
}

output "db_endpoint" {
  description = "Endpoint de conexión a la base de datos"
  value       = aws_db_instance.main.endpoint
}
```

### `terraform.tfvars` — valores del ambiente

```hcl
aws_region    = "us-east-1"
environment   = "production"
instance_type = "t3.small"
# db_password se pasa por variable de entorno: TF_VAR_db_password=...
```

### Comandos esenciales

```bash
# Inicializar (descarga providers y configura backend)
terraform init

# Validar la sintaxis
terraform validate

# Ver qué va a cambiar (sin aplicar nada)
terraform plan -var-file=terraform.tfvars

# Aplicar los cambios
terraform apply -var-file=terraform.tfvars

# Ver el estado actual
terraform show

# Eliminar toda la infraestructura
terraform destroy -var-file=terraform.tfvars

# Formatear los archivos .tf
terraform fmt -recursive
```

---

## ✅ Cuándo usar Terraform

| Escenario | Por qué Terraform gana |
|-----------|----------------------|
| Infraestructura en cloud (AWS, GCP, Azure) | Crea y gestiona cualquier recurso cloud desde código |
| Múltiples ambientes (dev/staging/prod) | Mismo código, diferentes `tfvars` — infraestructura idéntica en cada ambiente |
| Trabajo en equipo sobre infraestructura | El estado en S3 + locks en DynamoDB evita conflictos |
| Necesitas auditar cambios de infraestructura | Los `.tf` en git dan historial completo de quién cambió qué y cuándo |
| Infraestructura que se crea y destruye frecuentemente | `apply` / `destroy` en segundos, sin hacer clic en consolas |
| Multi-cloud o uso de servicios de terceros (Cloudflare, Datadog) | Un solo flujo de trabajo para todos los providers |

> **No es la mejor opción** para gestionar la configuración interna de los servidores (instalar paquetes, editar archivos) — para eso son mejores **Ansible** o los **user data scripts**. Terraform provisiona infraestructura, no la configura internamente.

---

## ↩️ Navegación
- [[Server/IaC/IaC|🏗️ IaC Hub]] → [[Server/Server|🖥️ Server]] → [[HOME|🏠 HOME]]
