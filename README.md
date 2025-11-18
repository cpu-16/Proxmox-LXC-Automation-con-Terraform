# 🚀 Proxmox LXC Automation con Terraform

<div align="center">

![Proxmox](https://img.shields.io/badge/Proxmox-VE%209.0.11-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-1.x-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)
![LXC](https://img.shields.io/badge/LXC-Containers-00ADD8?style=for-the-badge&logo=linuxcontainers&logoColor=white)

**Automatiza la creación y destrucción de contenedores LXC en Proxmox VE desde Windows** 🪟

</div>

---

## 📋 Tabla de Contenidos

- [Objetivo](#-objetivo)
- [Requisitos Previos](#-requisitos-previos)
- [Configuración de Proxmox](#-configuración-de-proxmox)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Archivos de Configuración](#-archivos-de-configuración)
- [Uso](#-uso)
- [Notas Importantes](#-notas-importantes)
- [Troubleshooting](#-troubleshooting)

---

## 🎯 Objetivo

Este proyecto automatiza la gestión de contenedores LXC en **Proxmox VE 9.0.11** usando **Terraform** desde Windows. Permite crear y destruir múltiples contenedores con configuración estandarizada de forma declarativa.

### Características de los contenedores

- **CPU:** 1 vCPU
- **RAM:** 512 MB
- **Disco:** 8 GB en `local-lvm`
- **Red:** Bridge `vmbr0` con DHCP
- **Nomenclatura:** `LXC-test-1`, `LXC-test-2`, etc.

---

## 📦 Requisitos Previos

### En Proxmox VE

- ✅ **Versión:** Proxmox VE 9.0.x (probado en 9.0.11)
- ✅ **Templates LXC** descargadas en `local:vztmpl/`:
  - `debian-12-standard_12.7-1_amd64.tar.zst`
  - `ubuntu-25.04-standard_25.04-1.1_amd64.tar.zst`

  ![Templates LXC en Proxmox](images/templates-lxc.png)

- ✅ **Bridge de red** configurado (`vmbr0` con DHCP)

### En Windows

- 🔧 **Terraform** instalado ([Download](https://www.terraform.io/downloads))
- 🔧 **Git** (opcional, para control de versiones)
- 🔧 **PowerShell** o terminal preferida

---

## 🔐 Configuración de Proxmox

### 1️⃣ Crear Usuario de Servicio

1. Navega a: **Datacenter → Permissions → Users → Add**
2. Configura:
   - **User name:** `terraform`
   - **Realm:** `Proxmox VE authentication server (pve)`
   - **Password:** Una contraseña fuerte
   - **Enabled:** ✅ Marcado

![Crear Usuario Terraform](images/crear-usuario.png)

> 💡 **Nota:** Usamos el realm `pve` (no Linux PAM) para un usuario nativo de Proxmox.

### 2️⃣ Crear Token de API

1. Ve a: **Datacenter → Permissions → API Tokens → Add**
2. Configura:
   - **User:** `terraform@pve`
   - **Token name:** `terraform1`
   - **Privilege Separation:** `No` (hereda permisos del usuario)
   - **Expire:** `never` (para laboratorio)

![Crear Token API](images/crear-token.png)

3. **¡IMPORTANTE!** Copia el **Secret** mostrado (solo se muestra una vez)

```
Token ID: terraform@pve!terraform1
Secret: 73cdccb3-d2b2-4fa0-879e-b1c7da3b6ya8
```

![Secret del Token](images/token-secret.png)

### 3️⃣ Asignar Permisos

1. Ve a: **Datacenter → Permissions → Add → User Permission**
2. Configura:
   - **Path:** `/` (root)
   - **User:** `terraform@pve`
   - **Role:** `Administrator`
   - **Propagate:** ✅ Marcado

![Asignar Permisos](images/asignar-permisos.png)

---

## 📁 Estructura del Proyecto

```
proxmox-lxc-clones/
├── 📄 main.tf              # Configuración principal
├── 📄 variables.tf         # Definición de variables
├── 📄 terraform.tfvars     # Valores secretos (NO subir a Git)
├── 📄 .gitignore           # Archivos a ignorar
├── 📄 README.md            # Esta documentación
└── 📁 images/              # Capturas de pantalla
    ├── crear-usuario.png
    ├── crear-token.png
    ├── token-secret.png
    ├── asignar-permisos.png
    ├── terraform-init.png
    ├── terraform-plan.png
    ├── terraform-apply.png
    ├── lxc-creados.png
    └── terraform-destroy.png
```

### Inicializar el Proyecto

```powershell
cd C:\Users\tu-usuario\Desktop\git
mkdir proxmox-lxc-clones
cd proxmox-lxc-clones
git init
```

---

## 📝 Archivos de Configuración

### `main.tf`

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "3.0.2-rc04"  # ⚠️ Versión probada y funcional
    }
  }
}

provider "proxmox" {
  pm_api_url          = var.pm_api_url
  pm_api_token_id     = var.pm_api_token_id
  pm_api_token_secret = var.pm_api_token_secret
  pm_tls_insecure     = true  # Para certificados self-signed
}

resource "proxmox_lxc" "test-container" {
  count = var.lxc_count

  hostname    = "LXC-test-${count.index + 1}"
  target_node = var.node_name
  vmid        = 2000 + count.index

  ostemplate = "local:vztmpl/${var.lxc_template}"
  cores      = var.lxc_cores
  memory     = var.lxc_memory
  password   = var.lxc_password

  unprivileged = true
  onboot       = true
  start        = true

  rootfs {
    storage = var.lxc_storage
    size    = "8G"
  }

  features {
    nesting = true
  }

  network {
    name   = "eth0"
    bridge = "vmbr0"
    ip     = "dhcp"
    type   = "veth"
  }
}
```

### `variables.tf`

```hcl
variable "pm_api_url" {
  type        = string
  description = "URL de la API de Proxmox"
}

variable "pm_api_token_id" {
  type        = string
  description = "Token ID del usuario terraform"
}

variable "pm_api_token_secret" {
  type        = string
  sensitive   = true
  description = "Secret del token de API"
}

variable "node_name" {
  type        = string
  default     = "ciber"
  description = "Nombre del nodo Proxmox"
}

variable "lxc_template" {
  type        = string
  default     = "debian-12-standard_12.7-1_amd64.tar.zst"
  description = "Template LXC a utilizar"
}

variable "lxc_storage" {
  type        = string
  default     = "local-lvm"
  description = "Storage pool para los contenedores"
}

variable "lxc_password" {
  type        = string
  default     = "Letmein1$"
  sensitive   = true
  description = "Password root de los LXC (solo para lab)"
}

variable "lxc_memory" {
  type        = number
  default     = 512
  description = "RAM en MB"
}

variable "lxc_cores" {
  type        = number
  default     = 1
  description = "Número de vCPUs"
}

variable "lxc_count" {
  type        = number
  default     = 5
  description = "Cantidad de contenedores a crear"
}
```

### `terraform.tfvars` 🔒

> ⚠️ **NUNCA subir este archivo a Git** (contiene credenciales)

```hcl
pm_api_url          = "https://10.3.33.53:8006/api2/json"
# O usando FQDN: "https://pve.ciberpty.com/api2/json"

pm_api_token_id     = "terraform@pve!terraform1"
pm_api_token_secret = "AQUI_VA_EL_SECRET_REAL_DEL_TOKEN"

node_name    = "ciber"
lxc_password = "Letmein1$"
```

### `.gitignore`

```gitignore
# Terraform
.terraform/
.terraform.lock.hcl
terraform.tfstate
terraform.tfstate.backup

# Secretos
terraform.tfvars

# Logs
crash.log
*.log
```

---

## 🎮 Uso

### 1️⃣ Inicializar Terraform

```powershell
terraform init
```

Esto descarga el provider `telmate/proxmox` versión `3.0.2-rc04`.

![Terraform Init](images/terraform-init.png)

### 2️⃣ Formatear Código (Opcional)

```powershell
terraform fmt
```

### 3️⃣ Validar Configuración

```powershell
terraform validate
```

### 4️⃣ Ver Plan de Ejecución

```powershell
terraform plan
```

**Salida esperada:**

```
Plan: 5 to add, 0 to change, 0 to destroy.
```

![Terraform Plan](images/terraform-plan.png)

### 5️⃣ Crear Contenedores

```powershell
terraform apply
```

Escribe `yes` para confirmar. Terraform creará 5 contenedores LXC (VMID 2000-2004).

![Terraform Apply](images/terraform-apply.png)

### 6️⃣ Verificar en Proxmox

En la GUI de Proxmox verás:

- **CT 2000** → `LXC-test-1`
- **CT 2001** → `LXC-test-2`
- **CT 2002** → `LXC-test-3`
- **CT 2003** → `LXC-test-4`
- **CT 2004** → `LXC-test-5`

![Contenedores Creados en Proxmox](images/lxc-creados.png)

### 7️⃣ Destruir Contenedores

```powershell
terraform destroy
```

Confirma con `yes` para eliminar todos los contenedores.

![Terraform Destroy](images/terraform-destroy.png)

#### Destruir un Contenedor Específico

```powershell
terraform destroy -target=proxmox_lxc.test-container[0]
```

---

## ⚠️ Notas Importantes

### Versión del Provider

| Versión | Creación | Destrucción | Estado |
|---------|----------|-------------|--------|
| `2.9.14` | ✅ | ⚠️ Errores 401/500 | ❌ No recomendada |
| `3.0.2-rc05` | ✅ | ❌ Error 500 | ❌ No recomendada |
| `3.0.2-rc04` | ✅ | ✅ | ✅ **RECOMENDADA** |

> 🎯 **Recomendación:** Mantener `version = "3.0.2-rc04"` en `main.tf`

### Seguridad

- 🔐 **Nunca** subas `terraform.tfvars` a repositorios públicos
- 🔐 Usa tokens con permisos mínimos en producción
- 🔐 Considera usar variables de entorno o vaults para secretos:

```powershell
$env:TF_VAR_pm_api_token_secret = "tu-secret-aqui"
```

### Templates

Verifica que los templates existan en:

**Proxmox GUI:** `Datacenter → Storage (local) → CT Templates`

---

## 🔧 Troubleshooting

### Error: 401 Authentication Failure

- ✅ Verifica que el token ID sea correcto: `terraform@pve!terraform1`
- ✅ Confirma que el secret esté bien copiado en `terraform.tfvars`
- ✅ Revisa los permisos del usuario en Proxmox

### Error: 500 Internal Server Error

- ✅ Usa la versión `3.0.2-rc04` del provider
- ✅ Verifica que el template exista en Proxmox
- ✅ Confirma que el storage `local-lvm` esté disponible

### Error: Template Not Found

```bash
# Verifica el nombre exacto del template
ls /var/lib/vz/template/cache/
```

El nombre debe coincidir exactamente con `lxc_template` en `variables.tf`.

### Los Contenedores No Inician

- ✅ Verifica que el bridge `vmbr0` esté configurado
- ✅ Confirma que haya espacio en `local-lvm`
- ✅ Revisa los logs en Proxmox

---

## 🤝 Contribuir

¿Mejoras o sugerencias? ¡Pull requests bienvenidos!

1. Fork el proyecto
2. Crea tu rama: `git checkout -b feature/nueva-funcionalidad`
3. Commit: `git commit -m 'Añade nueva funcionalidad'`
4. Push: `git push origin feature/nueva-funcionalidad`
5. Abre un Pull Request

---

## 📄 Licencia

Este proyecto es libre de usar para propósitos educativos y de laboratorio.

---

## 🙏 Agradecimientos

- [Telmate Proxmox Provider](https://github.com/Telmate/terraform-provider-proxmox)
- [Proxmox VE](https://www.proxmox.com/)
- [Terraform by HashiCorp](https://www.terraform.io/)

---

<div align="center">

**⭐ Si te resultó útil, dale una estrella al repo! ⭐**

Hecho con ❤️ para homelabs y aprendizaje

</div>
