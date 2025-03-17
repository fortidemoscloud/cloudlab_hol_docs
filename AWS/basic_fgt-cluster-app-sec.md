# Despligue de cluster de FortiGates en HA

Este repositorio despliega una arquitectura en AWS de una VPC en dos Zonas de Disponibildiad (AZs) de un cluster FortiGate y una instancia virtual de kubernetes con dos aplicaciones desplegadas para testing.

Esta basado en el módulo de [AWS Terraform Registry](https://registry.terraform.io/modules/jmvigueras/ftnt-aws-modules/aws/latest/examples/basic_fgt-cluster)

> [!NOTE] 
> Este módulo permite escoger el número de AZs a desplegar y el número de FortiGates a desplegar por AZ. Para despliegues Activo-Pasivo el número de FortiGates debe ser de 2. 

## Deployment Overview

- Una VPC con estas subnets: Management y HA, Public, Private, Bastion y TGW. 
- Fortigate cluster: 2 instancias de 3 interfaces, con un interfaz de Management con una IP pública asociada para gestión y funcionamiento del conector SDN para HA. Un interfaz público con una IP pública asociada al FortiGate activo y un interfaz interno. 
- Una instancia Linux con un entorno de Kubernetes con 2 aplicaciones desplegadas para testing: DVWA y API Swagger. 

## Diagrama de arquitectura

![FortiGate reference architecture overview](images/basic_fgt-ha-xlb-app-sec_01.png)

## Código Terraform para despligue fuera de CloudLab Portal

```hcl

# FGT cluster FGCP in 1 AZ with 2 members

# Define custom_vars at terraform.tfstate or use terraform cli -var option. 
custom_vars = {
    prefix                     = "fgt-appsec"
    region                     = "eu-west-1"
    fgt_build                  = "build2731"
    license_type               = "payg"
    fgt_size                   = "c6i.large"
    fgt_cluster_type           = "fgcp"
    fgt_number_peer_az         = 1
    number_azs                 = 2
    fgt_vpc_cidr               = "172.10.0.0/23"
    public_subnet_names_extra  = ["bastion"]
    private_subnet_names_extra = ["protected"]
    k8s_size                   = "t3.2xlarge"
    k8s_version                = "1.31"
    tags                       = { "Deploy" = "CloudLab AWS", "Project" = "CloudLab" }
}

# FGT cluster module
module "fgt-cluster" {
  source  = "jmvigueras/ftnt-aws-modules/aws//examples/basic_fgt-cluster"
  version = "0.0.12"

  prefix = var.custom_vars["prefix"]

  region = var.custom_vars["region"]
  azs    = local.azs

  fgt_build     = var.custom_vars["fgt_build"]
  license_type  = var.custom_vars["license_type"]
  instance_type = var.custom_vars["fgt_size"]

  fgt_number_peer_az = var.custom_vars["fgt_number_peer_az"]
  fgt_cluster_type   = var.custom_vars["fgt_cluster_type"]

  fgt_vpc_cidr               = var.custom_vars["fgt_vpc_cidr"]
  public_subnet_names_extra  = var.custom_vars["public_subnet_names_extra"]
  private_subnet_names_extra = var.custom_vars["private_subnet_names_extra"]
}
```

### Requirements
* [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html) >= 1.6.0
* Check particulars requiriments for each deployment (AWS) 


## Soporte
Este es un repositorio personal de despliegues y testeo de soluciones Fortinet en Cloud. No tiene ningún tipo de soporte y debe usarse bajo tu propia responsabilidad. Los proveedores Cloud pueden añadir cargos por estos despliegues, por lo que debes tenerlo en cuenta antes de continuar. 