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
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Name: **pubSubnetA**
  - Availability Zone (AZ): **az1**
  - IPv4 CIDR block: **10.0.1.0/24**
  - **Create**
- Repita os passos acima para criar as demais Subnets

| Name | AZ | IPv4 CIDR |
| --- | --- | --- |
| **pubSubnetB** | **az2** | **10.0.11.0/24** |
| **privSubnetA** | **az1** | **10.0.2.0/24** |
| **privSubnetB** | **az2** | **10.0.12.0/24** |

- Selecione **pubSubnetA**, clique em "**Action > Edit subnet settings**", va em "**Auto-assign IP settings**" e escolha "**Enable auto-assign public IPv4 address**". Repita o mesmo processo com **pubSubnetB**.

### Passo 03: Internet Gateway

- VPC > Internet Gateways > Create internet gateway
  - Name: **igw-mdc**
  - **Create**
- Atachar o Internet Gateway na VPC
  - VPC > Internet Gateways > Actions > Attach to VPC [**vpcMDC**]

### Passo 04: Criar as Route Tables (pub/priv)

**Public Route Table:**

- VPC > Route Tables > Create route table
  - Name: **rt-public**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - **Create**
- Inserção de uma rota para o Internet Gateway (comunicação com a Internet)
  - Route Tables > [**rt-public**] > Routes > Edit routes > Add route: **0.0.0.0/0** com Target no Internet Gateway [**igw-mdc**]
- Associação com as Subnets públicas
  - Route Tables > [**rt-public**] > Subnet associations -> Edit subnet associations:
    - selecione as Subnets públicas: [**pubSubnetA**]; [**pubSubnetB**]

**Private Route Table:**

- VPC > Route Tables > Create route table
  - Name: **rt-private**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - **Create**
- Associação com as Subnets privadas
  - Route Tables > [**rt-private**] > Subnet associations > Edit subnet associations:
    - selecione as Subnets privadas: [**privSubnetA**]; [**privSubnetB**]

### VPC Resource Map

![vpc-resource-map](./img/vpc-resource-map.png)

## Elastic Load Balancer (NLB): implantação com 2 instâncias em alta disponibilidade

**Desenho da Arquitetura**:

![NLB-diagram](./img/Diagram-NLB.png)

### Passo 01: Criar uma Key Pair

- EC2 > Network & Security > Key Pairs > Create key pair
  - Name: **keypair**
  - Key pair type: **RSA**
  - Private key file format: **.pem**
  - **Create**

### Passo 02: Criar Security Groups

- EC2 > Network & Security > Security Groups > Create security group
  - Name: **sgELB**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Inbound Rules > Add Rule:

    | Type | Source | Description |
    | --- | --- | --- |
    | HTTP (80) | Anywhere-IPv4 | Allow HTTP |
    | HTTPS (443) | Anywhere-IPv4 | Allow HTTPS |
  
  - **Create**

- EC2 > Network & Security > Security Groups > Create security group
  - Name: **sgWebServer**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Inbound Rules > Add Rule:

    | Type | Source | Description |
    | --- | --- | --- |
    | SSH (22) | Anywhere-IPv4 | Allow SSH |
    | HTTP (80) | Anywhere-IPv4 | Allow HTTP |
    | All TCP | Custom: [sgELB] | Allow Load Balancer |

  - **Create**

### Passo 03: Criar duas instâncias EC2 (Elastic Compute Cloud)

- EC2 > Instances > Instances > Launch instances
  - Name: **WebServer1**; repertir passos para **WebServer2**
  - Amazon Machine Image (AMI): **Amazon Linux**
  - Instance type: **t2.micro**
  - Key pair: **keypair**
  - Network settings > Edit:
    - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
    - Subnet: WebServer1 [**pubSubnetA**]; WebServer2 [**pubSubnetB**]
    - Select existing security group > Security Groups: [**sgWebServer**]
  - Advanced details > UserData:

    ```bash
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello World from Load Balancer $(hostname -f)</h1>" > /var/www/html/index.html
    ```

  - **Launch instance**

- para testar a conexão na Instância pública via SSH, usar o comando:

    ```bash
    ssh -i [nomekeypair.pem] ec2-user@[PublicIP]
    ```

### Passo 04: Criar um Load Balancer (NLB)

- EC2 > Load Balancing > Load balancers > Create load balancer
  - Load balancer types: **Network Load Balancer** > Create
  - Name: **nlb-mdc**
  - Scheme: **Internet-facing**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Mappings (AZs / Subnets):
    - az1: [**pubSubnetA**]
    - az2: [**pubSubnetB**]
  - Security Groups: [**sgELB**]
  - Listeners **TCP:80** > Create target group:
    - Target type: **Instances**
    - Name: **tg-nlb-mdc**
    - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
    - Health checks > Advanced health check settings: **Timeout: 2**; **Interval: 5**
    - **Next**
    - Available instances: [**WebServer1**] ; [**WebServer2**] > **Include as pending below**
    - **Create target group**
  - Forward to: selecionar o Target group criado [**tg-nlb-mdc**]
  - **Create load balancer**

### Passo 05: Validar Network Load Balancer

- Aguardar o provisionamento e inicialização do Load Balancer
- Aguardar que o Target group registre as instâncias EC2 e atinja o estado "healthy"
- Copiar o "**DNS name**" do Loab Balancer e testar no web browser

## Aplication Load Balancer (ALB) com Auto Scaling Group (ASG)

**Desenhho da Arquitetura**:

![ALB-diagram](./img/Diagram-ALB.png)

### Passo 01: Criar Security Groups

- EC2 > Network & Security > Security Groups > Create security group
  - Name: **sgASG**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Inbound Rules > Add Rule:

    | Type | Source | Description |
    | --- | --- | --- |
    | HTTP (80) | Anywhere-IPv4 | Allow HTTP |
    | HTTPS (443) | Anywhere-IPv4 | Allow HTTPS |
    | All TCP | Custom: [sgELB] | Allow Load Balancer |

  - **Create**

### Passo 02: Criar um Aplication Load Balancer (ALB)

- EC2 > Load Balancing > Load balancers > Create load balancer
  - Load balancer types: **Application Load Balancer** > Create
  - Name: **alb-mdc**
  - Scheme: **Internet-facing**
  - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
  - Mappings (AZs / Subnets):
    - az1: [**pubSubnetA**]
    - az2: [**pubSubnetB**]
  - Security Groups: [**sgELB**]
  - Listeners **HTTP:80** > Create target group:
    - Target type: **Instances**
    - Name: **tg-alb-mdc**
    - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
    - Health checks > Advanced health check settings: **Timeout: 2**; **Interval: 5**
    - **Next**
    - Available instances: não selecionar as instâncias - deixar vazio
    - **Create target group**
  - Forward to: selecionar o Target group criado [**tg-alb-mdc**]
  - **Create load balancer**

### Passo 03: Criar um Auto Scaling Group (ASG)

- EC2 > Auto Scaling > Auto Scaling Groups > Create Auto Scaling group
  - Name: **asg-mdc**
  - Launch template > Create a launch template:
    - Template name: **template-asg-mdc**
    - Template version description: **First template**
    - Amazon Machine Image (AMI) > Quick Start: **Amazon Linux**
    - Instance type: **t2.micro**
    - Key pair: **keypair**
    - Network settings:
      - Subnet: **Don't include in launch template**
      - Select existing security group > Security Groups: [**sgASG**]
    - Advanced details > UserData:

      ```bash
      #!/bin/bash
      yum update -y
      yum install -y httpd
      systemctl start httpd
      systemctl enable httpd
      echo "<h1>Hello World from Application Load Balancer $(hostname -f)</h1>" > /var/www/html/index.html
      ```

    - **Create Launch template**
  - Launch template: selecionar Launch template criado [**template-asg-mdc**]
  - **Next**
  - Network:
    - VPC: selecionar a VPC criada anteriormente [**vpcMDC**]
    - AZs and subnets: [**pubSubnetA**]; [**pubSubnetB**]
  - **Next**
  - Load balancing: **Attach to an existing load balancer**
  - Choose from your load balancer target groups > target groups: [**tg-alb-mdc**]
  - **Next**
  - Group size / Scaling:
    - Desired capacity: 2
    - Min desired capacity: 2
    - Max desired capacity: 4
  - **Next**
  - **Next**
  - **Next**
  - **Create Auto Scaling group**

### Passo 04: Validar Application Load Balancer

- Aguardar o provisionamento e inicialização do Application Load Balancer
- Aguardar que o ASG inicialize e o Target group registre as instâncias EC2 e atinja o estado "healthy"
- Copiar o "**DNS name**" do Application Loab Balancer e testar no web browser

![alb-10-0-01](./img/alb-10-0-01.png)

![alb-10-0-11](./img/alb-10-0-11.png)

## AWS Resource Groups

**Recursos criados ao final do Laboratório:**

![resource-group](./img/resource-group.png)

##

Bons estudos!!!

**André Carlucci**
