# Documentação - Infraestrutura DevSecOps na AWS

## Introdução

Este projeto tem como objetivo a criação de uma infraestrutura na AWS utilizando boas práticas de DevSecOps. A arquitetura contempla:

- VPC
- RDS (PostgreSQL)
- EFS
- EC2 com Auto Scaling Group e Launch Template
- Security Groups
- Load Balancer

---

## 1. Criação da VPC

Criamos uma VPC personalizada com:

- CIDR block: `10.0.0.0/16`
- Duas subnets públicas e duas privadas em zonas de disponibilidade distintas
- Internet Gateway (IGW) anexado à VPC
- Tabela de roteamento com rota para a internet associada às subnets públicas
- Subnets privadas configuradas para acessar a internet via NAT Gateway

**O que apareceria na imagem:**
> Tela da VPC mostrando as subnets públicas e privadas criadas e o roteamento associado.

---

## 2. Banco de Dados RDS

Criamos um banco de dados PostgreSQL na AWS RDS com:

- Nome do banco: `devsecopsdb`
- Subnet group configurado com as subnets privadas
- Acesso restrito via Security Group apenas para a aplicação na EC2
- Backups habilitados

**O que apareceria na imagem:**
> Tela do RDS com detalhes da instância PostgreSQL e configurações de rede.

---

## 3. EFS (Elastic File System)

O EFS foi configurado para armazenamento compartilhado entre instâncias EC2:

- Criado na mesma VPC
- Subnet group com as subnets privadas
- Permissões via Security Group apenas para instâncias EC2

**O que apareceria na imagem:**
> Tela do EFS mostrando os mount targets e configurações de segurança.

---

## 4. EC2 + Launch Template + Auto Scaling Group

Criamos:

- Um **Launch Template** com AMI customizada contendo o Docker
- Script de userdata que roda o container da aplicação automaticamente
- Um **Auto Scaling Group** com duas instâncias mínimas
- As instâncias utilizam o EFS como volume adicional
- As instâncias estão em subnets privadas

**Userdata (exemplo):**
```bash
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
systemctl enable docker
docker run -d -p 80:80 minhaimagemdevsecops
