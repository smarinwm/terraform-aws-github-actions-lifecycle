# Gestión de infraestructura AWS con Terraform y GitHub Actions

Proyecto de práctica orientado a **Infrastructure as Code (IaC)** con **Terraform**, **Amazon Web Services (AWS)** y **GitHub Actions**.

El repositorio permite desplegar y destruir una instancia **Amazon EC2** mediante workflows manuales de GitHub Actions, utilizando secretos del repositorio para las credenciales de AWS y almacenando el estado de Terraform en **Amazon S3** en algunos de los flujos disponibles.

## Objetivo del proyecto

El propósito principal es practicar:

- Aprovisionamiento de infraestructura con Terraform.
- Creación de instancias EC2.
- Parametrización mediante variables.
- Outputs de Terraform.
- Automatización de `terraform apply`.
- Automatización de `terraform destroy`.
- Uso de GitHub Actions.
- Gestión de credenciales mediante GitHub Secrets.
- Persistencia y recuperación del estado de Terraform desde Amazon S3.
- Creación de acciones compuestas reutilizables.

## Tecnologías utilizadas

- **Terraform**
- **Amazon Web Services (AWS)**
- **Amazon EC2**
- **Amazon S3**
- **GitHub Actions**
- **YAML**
- **AWS CLI**
- **Bash**
- **Infrastructure as Code (IaC)**
- **CI/CD**

## Estructura del repositorio

```text
terra-destroy/
├── main.tf
├── variables.tf
├── outputs.tf
├── action.yml
├── .github/
│   └── workflows/
│       ├── deploy.yml
│       ├── deploy2.yml
│       ├── destroy3.yml
│       └── terraform-destroy.yml
└── README.md
```

## Infraestructura Terraform

### `main.tf`

Define el provider de AWS y una instancia EC2:

```hcl
provider "aws" {
  region = var.aws_region
}

resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name = "example-instance-smm"
  }
}
```

La configuración utiliza variables para evitar escribir directamente en el recurso valores como región, AMI o tipo de instancia.

## Variables

`variables.tf` define:

```text
aws_region
ami_id
instance_type
```

Los valores por defecto incluidos son:

```text
Región:       us-east-1
Instancia:    t3.micro
AMI:          configurable mediante ami_id
```

Esto permite modificar el despliegue sin alterar el recurso principal.

## Outputs

El proyecto devuelve información útil después del despliegue:

```text
instance_public_ip
instance_id
```

De esta forma se puede recuperar automáticamente la IP pública y el identificador de la EC2 creada.

## Automatización con GitHub Actions

El repositorio contiene varias versiones de workflows creadas como ejercicios de evolución del proceso de automatización.

### `deploy.yml`

Workflow manual para:

```text
Checkout
   ↓
Configurar credenciales AWS
   ↓
Instalar Terraform
   ↓
terraform init
   ↓
terraform plan
   ↓
terraform apply
```

Se ejecuta mediante:

```yaml
workflow_dispatch:
```

por lo que el despliegue debe iniciarse manualmente desde GitHub Actions.

### `terraform-destroy.yml`

Workflow independiente para destruir la infraestructura:

```text
Checkout
   ↓
Configurar AWS
   ↓
Instalar Terraform
   ↓
terraform init
   ↓
terraform destroy
```

También utiliza ejecución manual.

### `deploy2.yml`

Esta versión reúne `apply` y `destroy` en un único workflow.

Al iniciarlo desde GitHub se puede seleccionar:

```text
Terraform_apply
Terraform_destroy
```

#### Flujo de `apply`

El workflow:

1. Inicializa Terraform.
2. Ejecuta `terraform apply -auto-approve`.
3. Copia `terraform.tfstate` a Amazon S3.
4. Copia `.terraform.lock.hcl` a S3.

#### Flujo de `destroy`

El workflow:

1. Inicializa Terraform.
2. Recupera `terraform.tfstate` desde S3.
3. Recupera `.terraform.lock.hcl`.
4. Genera el plan de destrucción.
5. Ejecuta `terraform destroy -auto-approve`.
6. Elimina de S3 los ficheros asociados cuando la destrucción finaliza correctamente.

Esto permite practicar la necesidad de conservar el estado de Terraform cuando las ejecuciones de CI se producen en runners efímeros.

## Acción compuesta

El archivo:

```text
action.yml
```

define una **GitHub Composite Action** reutilizable.

Centraliza tareas como:

- Configuración de credenciales AWS.
- Instalación de Terraform.
- Ejecución de `terraform init`.

El workflow `destroy3.yml` utiliza esta acción local mediante:

```yaml
uses: ./
```

Esto reduce duplicación y permite practicar la reutilización de pasos comunes en GitHub Actions.

## GitHub Secrets necesarios

Los workflows esperan secretos con nombres como:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_DEFAULT_REGION
BUCKET
```

Estos valores deben configurarse en GitHub y **no deben almacenarse directamente en el repositorio**.

Si se utilizan credenciales temporales de AWS Academy, laboratorios u otros entornos similares, el `AWS_SESSION_TOKEN` también debe actualizarse cuando caduque.

## Uso local con Terraform

### Requisitos

- Terraform instalado.
- Una cuenta AWS.
- Credenciales AWS configuradas.
- Permisos para crear y destruir instancias EC2.

### Clonar el repositorio

```bash
git clone https://github.com/smarinwm/terra-destroy.git
cd terra-destroy
```

### Inicializar Terraform

```bash
terraform init
```

### Revisar el despliegue

```bash
terraform plan
```

### Crear la infraestructura

```bash
terraform apply
```

### Consultar outputs

```bash
terraform output
```

### Destruir los recursos

```bash
terraform destroy
```

## Uso desde GitHub Actions

Los workflows utilizan `workflow_dispatch`, por lo que se ejecutan manualmente desde:

```text
GitHub
→ Actions
→ Seleccionar workflow
→ Run workflow
```

Para `deploy2.yml` o `destroy3.yml`, selecciona la operación correspondiente:

```text
Terraform_apply
```

o:

```text
Terraform_destroy
```

## Estado de Terraform

Una parte importante del proyecto es la gestión de:

```text
terraform.tfstate
```

Los runners de GitHub Actions son efímeros, de modo que el estado generado en una ejecución no estará disponible automáticamente en la siguiente.

El proyecto experimenta con **Amazon S3** para conservar ese estado entre las operaciones `apply` y `destroy`.

### Mejora recomendada

En un entorno real sería preferible configurar directamente un **backend remoto de Terraform en S3**, en lugar de copiar manualmente `terraform.tfstate` mediante AWS CLI.

Por ejemplo:

```hcl
terraform {
  backend "s3" {
    bucket = "mi-bucket-terraform"
    key    = "terra-destroy/terraform.tfstate"
    region = "us-east-1"
  }
}
```

La configuración concreta no debería incluir secretos.

Según las necesidades del entorno, también conviene estudiar mecanismos de bloqueo del estado para evitar modificaciones concurrentes.

## Aspectos que conviene mejorar

El repositorio refleja distintas pruebas y versiones del mismo flujo, algo útil durante el aprendizaje, pero antes de convertirlo en una referencia más limpia sería recomendable:

- Elegir un único workflow principal de `apply/destroy`.
- Retirar workflows antiguos o redundantes.
- Actualizar las versiones de las GitHub Actions utilizadas.
- Definir `required_version` de Terraform.
- Definir explícitamente la versión del provider AWS.
- Utilizar un backend remoto de Terraform.
- Evitar copiar `.terraform.lock.hcl` a S3 como parte del estado.
- Versionar `.terraform.lock.hcl` en Git cuando corresponda.
- Resolver la AMI de forma dinámica en lugar de depender de un ID fijo.
- Añadir tags como proyecto, entorno y propietario.
- Incorporar `terraform fmt` y `terraform validate` al pipeline.
- Valorar `terraform plan` como paso obligatorio antes de `apply`.

## AMI

Actualmente el proyecto define una AMI por defecto mediante:

```hcl
variable "ami_id"
```

Los IDs de AMI dependen de la región y pueden quedar obsoletos.

Para mejorar la portabilidad se puede obtener una AMI actual mediante un `data source`, por ejemplo filtrando una imagen oficial de Amazon Linux.

## Seguridad

Aunque las credenciales se obtienen mediante GitHub Secrets, para un entorno real conviene reforzar el modelo de autenticación.

Recomendaciones:

- No almacenar Access Keys en código.
- Aplicar permisos mínimos a la identidad utilizada por Terraform.
- Rotar credenciales periódicamente.
- Evitar credenciales de larga duración.
- Valorar autenticación federada de GitHub Actions hacia AWS mediante **OpenID Connect (OIDC)**.
- Proteger el bucket que almacena el estado.
- Activar cifrado y versionado del bucket de estado.
- Restringir quién puede ejecutar workflows de despliegue y destrucción.
- Revisar siempre qué recursos va a destruir Terraform.

### GitHub Actions + AWS OIDC

Una evolución especialmente recomendable sería sustituir:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
```

por autenticación mediante **GitHub OIDC** y un rol IAM de AWS.

Esto reduce la necesidad de mantener credenciales AWS almacenadas como secretos de larga duración.

## Costes

Este repositorio puede crear recursos reales en AWS.

Aunque una instancia `t3.micro` puede estar cubierta en determinados escenarios por ofertas gratuitas o créditos, **no debe asumirse que su ejecución sea gratuita**.

Después de las pruebas:

```bash
terraform destroy
```

y comprueba también desde la consola de AWS que no permanezcan recursos activos.

## Objetivo didáctico

Este proyecto permite practicar conceptos de:

- Terraform.
- Infrastructure as Code.
- AWS EC2.
- Variables y outputs.
- Ciclo `init → plan → apply → destroy`.
- GitHub Actions.
- Workflows manuales.
- GitHub Composite Actions.
- GitHub Secrets.
- AWS CLI.
- Gestión del estado de Terraform.
- Amazon S3.
- Automatización de despliegues y destrucción.
- Fundamentos de CI/CD para infraestructura.

## Perfil

**Silverio Marín** — Docente TIC en Valencia, especializado en cloud computing, automatización e infraestructura como código.

Más contenidos sobre **cloud computing, AWS, Terraform y automatización**:

**[silveriomarin.com/cloud](https://silveriomarin.com/cloud/)**

GitHub: **[@smarinwm](https://github.com/smarinwm)**
