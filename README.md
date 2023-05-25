#### PROJETO DE COMPUTAÇÃO EM NUVEM
#### Insper - Computação em Nuvem - 2023.1
#### Por Thiago Shiguero Kawahara

Rubrica atingida: C+ (Ambiente funcionando com Fargate, ECS e documentação.)

---

# Deploy de um container com AWS ECS e Fargate usando Terraform

O AWS ECS é um serviço de orquestração de contêineres projetado para simplificar a implantação e o gerenciamento de aplicativos em contêineres na nuvem.

O AWS Fargate é um serviço que permite executar contêineres sem servidores, ou seja, permite a execução de contêineres sem a necessidade de provisionar ou gerenciar diretamente a infraestrutura.

Logo, o conjunto desses dois serviços da AWS é uma solução completa para o gerenciamento e a execução de aplicativos em contêineres na nuvem.

![image](https://github.com/thiagosk/AWS-ECS-Fargate/assets/71990438/b8421d4f-ab43-4f0a-b8a6-5266d35d7503)
  
Este roteiro é dividido em 4 partes:

1. Instalando terraform e configurando com AWS
2. Criando AWS ECS e Fargate
3. Executando Terraform
4. Verificando os recursos criados

**Nota: este roteiro foi executado em Ubuntu 22.04**

## Parte 1: Instalando terraform e configurando com AWS

### Instalando o Terraform

Rode o seguinte comando no terminal para instalar terraform:

```
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform
```

### Configurando o terraform com AWS

Rode os seguintes comandos no terminal:

```
sudo apt install awscli
```

Para acessar sua conta será necessário fornecer as suas credenciais no seguinte comando:

```
aws configure
```

Em *AWS Access Key ID* e *AWS Secret Access Key ID*: adicione o ID das suas chaves AWS.

Em *Default region name*: adicione uma região (como `us-east-1`).

Em *Default output format*: adicione um formato (como `json`).

Para verificar se a configuração foi bem sucedida, veja o arquivo *credentials* pelo seguinte comando: 

```
cat ~/.aws/credentials
```

## Parte 2: Criando AWS ECS e Fargate

Crie uma pasta chamada *terraform*. E dentro desta pasta os arquivos de configuração *network.tf*, *provider.tf*, *security.tf*, *ecs.tf*, *output.tf* e *lb.tf*:

```
mkdir terraform
cd terraform
touch vpc.tf provider.tf security.tf ecs.tf output.tf lb.tf
```

Iremos definir o provedor da *aws*. Você pode escolher a região que preferir, eu escolhi a "us-east-1".

Em *provider.tf* adicione:

```
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}
    
provider "aws" {
  region = "us-east-1"
}
```

Em *network.tf* adicione:

```
data "aws_availability_zones" "available_zones" {
  state = "available"
}

resource "aws_vpc" "default" {
  cidr_block = "10.32.0.0/16"
}

resource "aws_subnet" "public" {
  count                   = 2
  cidr_block              = cidrsubnet(aws_vpc.default.cidr_block, 8, 2 + count.index)
  availability_zone       = data.aws_availability_zones.available_zones.names[count.index]
  vpc_id                  = aws_vpc.default.id
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private" {
  count             = 2
  cidr_block        = cidrsubnet(aws_vpc.default.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available_zones.names[count.index]
  vpc_id            = aws_vpc.default.id
}

resource "aws_internet_gateway" "gateway" {
  vpc_id = aws_vpc.default.id
}

resource "aws_route" "internet_access" {
  route_table_id         = aws_vpc.default.main_route_table_id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.gateway.id
}

resource "aws_eip" "gateway" {
  count      = 2
  vpc        = true
  depends_on = [aws_internet_gateway.gateway]
}

resource "aws_nat_gateway" "gateway" {
  count         = 2
  subnet_id     = element(aws_subnet.public.*.id, count.index)
  allocation_id = element(aws_eip.gateway.*.id, count.index)
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.default.id

  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = element(aws_nat_gateway.gateway.*.id, count.index)
  }
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = element(aws_subnet.private.*.id, count.index)
  route_table_id = element(aws_route_table.private.*.id, count.index)
}
```

Em *security.tf* adicione:

```
resource "aws_security_group" "lb" {
  name        = "alb-security-group-exemplo"
  vpc_id      = aws_vpc.default.id

  ingress {
    protocol    = "tcp"
    from_port   = 80
    to_port     = 80
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "hello_world_task" {
  name        = "task-security-group-exemplo"
  vpc_id      = aws_vpc.default.id

  ingress {
    protocol        = "tcp"
    from_port       = 3000
    to_port         = 3000
    security_groups = [aws_security_group.lb.id]
  }

  egress {
    protocol    = "-1"
    from_port   = 0
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Em *lb.tf* adicione:

```
resource "aws_lb" "default" {
  name            = "lb-exemplo"
  subnets         = aws_subnet.public.*.id
  security_groups = [aws_security_group.lb.id]
}

resource "aws_lb_target_group" "hello_world" {
  name        = "target-group-exemplo"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.default.id
  target_type = "ip"
}

resource "aws_lb_listener" "hello_world" {
  load_balancer_arn = aws_lb.default.id
  port              = "80"
  protocol          = "HTTP"

  default_action {
    target_group_arn = aws_lb_target_group.hello_world.id
    type             = "forward"
  }
}
```

Em *ecs.tf* adicione:

```
resource "aws_ecs_task_definition" "hello_world" {
  family                   = "hello-world-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 1024
  memory                   = 2048

  container_definitions = <<DEFINITION
[
  {
    "image": "registry.gitlab.com/architect-io/artifacts/nodejs-hello-world:latest",
    "cpu": 1024,
    "memory": 2048,
    "name": "hello-world-app",
    "networkMode": "awsvpc",
    "portMappings": [
      {
        "containerPort": 3000,
        "hostPort": 3000
      }
    ]
  }
]
DEFINITION
}

resource "aws_ecs_cluster" "main" {
  name = "cluster-ecs-exemplo"
}

resource "aws_ecs_service" "hello_world" {
  name            = "hello-world-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.hello_world.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    security_groups = [aws_security_group.hello_world_task.id]
    subnets         = aws_subnet.private.*.id
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.hello_world.id
    container_name   = "hello-world-app"
    container_port   = 3000
  }

  depends_on = [aws_lb_listener.hello_world]
}
```

Em *output.tf* adicione:

```
output "load_balancer_ip" {
  value = aws_lb.default.dns_name
}
```

## Parte 3: Executando Terraform

Dentro da pasta *terraform* rode os seguinte comandos no terminal:

Para definir a infraestrutura e configurar o ambiente:

```
terraform init
```

Para examinar o código e planejar as alterações:

```
terraform plan
```

Para aplicar as alterações no ambiente, criando, atualizando ou removendo recursos.

```
terraform apply
```

Para remover todos os recursos criados nesse terraform:

```
terraform destroy
```

## Parte 4: Verificando os recursos criados

Cluster ECS:
![image](https://github.com/thiagosk/AWS-ECS-Fargate/assets/71990438/676caad3-8efe-484a-932e-94703e6a06d3)

Task rodando com Fargate:
![image](https://github.com/thiagosk/AWS-ECS-Fargate/assets/71990438/8fd81994-77b6-48b9-b6ea-20c23466ffd2)

Load balancer:
![image](https://github.com/thiagosk/AWS-ECS-Fargate/assets/71990438/d3e1a53d-d93b-44e8-907e-5d0eb3398384)

Aplicação do contêiner no navegador:
![image](https://github.com/thiagosk/AWS-ECS-Fargate/assets/71990438/2fb36950-a87b-44ae-8e92-faf728b8610e)

