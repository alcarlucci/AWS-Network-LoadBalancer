MDC Workshop

# Provisionamento de uma Arquitetura básica de Redes (VPC) e Load Balancer (NLB; ALB+ASG) na AWS

## AWS Networking

**Desenho da Arquitetura**:

![network-diagram](./img/Diagram-Network.png)

### Passo 01: Criar uma VPC (Virtual Private Cloud)

- VPC > Your VPCs > Create VPC
  - Name: **vpcMDC**
  - IPv4 CIDR block: **10.0.0.0/16**
  - **Create**

### Passo 02: Criar as Subnets (pub/priv)

Quatro Subnets serão criadas: 2 públicas e 2 privadas

- VPC > Subnets > Create subnet
  - Name: **pubSubnetA**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Availability Zone (AZ): **az1**
  - IPv4 CIDR block: **10.0.1.0/24**
  - **Create**
- Repita os passos acima para criar as demais Subnets

| Name | AZ | IPv4 CIDR |
| --- | --- | --- |
| **pubSubnetB** | **az2** | **10.0.11.0/24** |
| **privSubnetA** | **az1** | **10.0.2.0/24** |
| **privSubnetB** | **az2** | **10.0.12.0/24** |

- Selecione **pubSubnetA**, clique em "**Edit subnet settings**" e clique em "**Auto-assign IP settings**" e escolha "**Enable auto-assign public IPv4 address**". Repita o mesmo processo com **pubSubnetB**.

### Passo 03: Internet Gateway

- VPC > Internet Gateways > Create internet gateway
  - Name: **igw-mdc**
- Atachar o Internet Gateway na VPC
  - VPC > Internet Gateways > Actions > Attach to VPC [**vpcMDC**]

### Passo 04: Criar as Route Tables (pub/priv)

**Public Route Table:**

- VPC -> Route Tables -> Create route table
  - Name: **rt-public**
- Inserção de uma rota para o Internet Gateway (comunicação com a Internet)
  - Route Tables -> Routes -> Edit routes -> Add route: **0.0.0.0/0** com Target no Internet Gateway [**igw-mdc**]
- Associação com as Subnets públicas
  - Route Tables -> Subnet associations -> Edit subnet associations:
    - selecione as Subnets públicas: [**pubSubnetA**]; [**pubSubnetB**]

**Private Route Table:**

- VPC -> Route Tables -> Create route table
  - Name: **rt-private**
- Associação com as Subnets privadas
  - Route Tables -> Subnet associations -> Edit subnet associations:
    - selecione as Subnets privadas: [**privSubnetA**]; [**privSubnetB**]

## Elastic Load Balancer (NLB): implantação com 2 instâncias em alta disponibilidade

**Desenho da Arquitetura**:

![NLB-diagram](./img/Diagram-NLB.png)

### Passo 01: Criar instâncias EC2 (Elastic Compute Cloud)

### Passo 02: Criar um Load Balancer (NLB)

### Passo 03: Validar Load Balancer

## Aplication Load Balancer (ALB) com Auto Scaling Group (ASG)

**Desenhho da Arquitetura**:

![ALB-diagram](./img/Diagram-ALB.png)

### Passo 01: Criar um Aplication Load Balancer (ALB)

### Passo 02: Criar um Auto Scaling Group (ASG)
