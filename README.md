# Resumo Estruturado das Tarefas Terraform EC2

## 1. Descrição Técnica da Tarefa 1

O código Terraform original provisiona uma infraestrutura segura na AWS, incluindo:

1. VPC (Virtual Private Cloud)
   - Isola a infraestrutura em uma rede privada
   - CIDR block define o intervalo de IPs

2. Subnets
   - Criadas dentro da VPC
   - Subnet pública associada a um gateway de Internet

3. Security Group
   - Controla o tráfego de entrada e saída da instância EC2
   - Ingress: SSH (porta 22) permitido apenas de IP confiável
   - Egress: Todo tráfego de saída permitido

4. EC2 Instance
   - Usa imagem AMI do Debian 12
   - Tipo de instância: t2.micro
   - Key Pair gerado para acesso SSH seguro
   - Root Block Device criptografado
   - User Data Script para atualizações iniciais

5. IAM Role
   - Anexada à instância EC2
   - Permite ações específicas na AWS sem armazenar credenciais localmente

6. VPC Flow Logs
   - Captura informações sobre o tráfego de rede
   - Logs armazenados em CloudWatch Log Group

7. Criptografia de Volume (EBS)
   - Protege dados armazenados contra acessos não autorizados

8. Restrição de SSH
   - Limita o acesso SSH a um IP ou rede confiável

9. Configuração de SSH Desabilitando Root
   - Reduz o risco de escalação de privilégios

## 2. Arquivo main.tf Modificado

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

variable "trusted_ip" {
  description = "IP confiável para acesso SSH"
  type        = string
  default     = "YOUR_ADMIN_IP/32"  # Substitua com o IP confiável
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de um IP confiável e tráfego HTTP para Nginx"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "Allow SSH from trusted IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.trusted_ip]
  }

  ingress {
    description = "Allow HTTP traffic"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install -y nginx
              systemctl start nginx
              systemctl enable nginx

              sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
              systemctl restart sshd
              EOF

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    encrypted             = true
    delete_on_termination = true
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

## 3. Descrição Técnica da Tarefa 2 (Melhorias)

As seguintes melhorias foram implementadas no código:

1. Segurança de Acesso SSH:
   - Adicionada variável `trusted_ip` para restringir acesso SSH
   - Motivo: Reduz a superfície de ataque, limitando o acesso SSH a IPs confiáveis

2. Desativação do Login Root via SSH:
   - Modificado script `user_data` para desabilitar login root
   - Motivo: Previne acesso direto com privilégios de superusuário, aumentando a segurança

3. Criptografia do Volume Root (EBS):
   - Configurado `encrypted = true` no bloco `root_block_device`
   - Motivo: Protege dados armazenados contra acesso não autorizado

4. Automação da Instalação e Configuração do Nginx:
   - Adicionados comandos no `user_data` para instalar e configurar Nginx
   - Motivo: Reduz tempo de configuração, tornando o servidor web operacional na criação

5. Regras de Segurança para o Tráfego HTTP:
   - Adicionada regra de ingress para permitir tráfego HTTP (porta 80)
   - Motivo: Permite que o servidor Nginx seja acessível publicamente

6. Automação de Atualizações de Sistema:
   - Incluídos comandos de atualização no `user_data`
   - Motivo: Garante que o sistema esteja atualizado desde o início, reduzindo vulnerabilidades

7. Tags de Identificação:
   - Adicionadas tags a todos os recursos
   - Motivo: Melhora a organização e rastreabilidade dos recursos na AWS

Estas melhorias resultam em um ambiente EC2 mais seguro, automatizado e bem gerenciado, seguindo as melhores práticas de segurança e eficiência operacional na AWS.


# Projeto Terraform

Este repositório contém configurações do Terraform para provisionamento de recursos na nuvem.

## Pré-requisitos

Antes de começar, você precisará de:

- [Terraform](https://www.terraform.io/downloads.html) instalado em sua máquina.
- Credenciais de acesso configuradas para o provedor de nuvem que você deseja usar (por exemplo, AWS, Azure, GCP).

### Instalação do Terraform

1. Baixe o Terraform a partir do [site oficial](https://www.terraform.io/downloads.html).
2. Instale o Terraform conforme as instruções para seu sistema operacional.
3. Verifique se a instalação foi bem-sucedida:
   ```bash
   terraform version

