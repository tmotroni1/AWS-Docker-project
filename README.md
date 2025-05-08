#!/bin/bash

# 1. Instalação via Script de Start Instance (user_data.sh)

# Atualizar pacotes e instalar Docker
apt-get update -y
apt-get install docker.io -y

# Iniciar o serviço Docker
systemctl start docker
systemctl enable docker

# Baixar a imagem do Wordpress
docker pull wordpress:latest

# Criar uma instância RDS com MySQL
aws rds create-db-instance \
  --db-instance-identifier wordpress-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --allocated-storage 20 \
  --master-username admin \
  --master-user-password <senha_do_banco> \
  --vpc-security-group-ids sg-xxxxxxxx \
  --no-publicly-accessible \
  --db-name wordpressdb

# Rodar o container do Wordpress
docker run -d -p 8080:80 --name wordpress --link db:mysql -e WORDPRESS_DB_PASSWORD=<senha_do_banco> wordpress

# 2. Efetuar Deploy de uma Aplicação Wordpress

# Criar o container Wordpress e conectá-lo ao RDS
docker run -d -p 8080:80 --name wordpress --link db:mysql -e WORDPRESS_DB_PASSWORD=<senha_do_banco> wordpress

# 3. Configuração do Serviço EFS AWS para Arquivos Estáticos do Wordpress

# Criar o EFS
aws efs create-file-system \
  --creation-token wordpress-efs \
  --performance-mode generalPurpose \
  --throughput-mode bursting

# Montar o EFS no container
mount -t efs <id_do_efs>:/ /var/www/html

# 4. Configuração do Serviço de Load Balancer AWS para a Aplicação Wordpress

# Criar Load Balancer
aws elb create-load-balancer \
  --load-balancer-name wordpress-lb \
  --listeners Protocol=HTTP,LoadBalancerPort=80,InstanceProtocol=HTTP,InstancePort=8080 \
  --subnets subnet-xxxxxxxx subnet-yyyyyyyy \
  --security-groups sg-xxxxxxxx

# Configurar o Health Check
aws elb configure-health-check \
  --load-balancer-name wordpress-lb \
  --health-check Target=HTTP:8080/wp-admin/install.php,Interval=30,UnhealthyThreshold=2,HealthyThreshold=2,Timeout=5

# 5. Pontos de Atenção

# Não utilizar IP público para saída do serviço Wordpress
# A aplicação não será exposta diretamente via IP público, apenas através do Load Balancer.

# 6. Dockerfile ou Docker Compose

# Usando Docker Compose (se preferir ao invés de Dockerfile)
echo "
version: '3'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - '8080:80'
    environment:
      WORDPRESS_DB_PASSWORD: <senha_do_banco>
    volumes:
      - wordpress_data:/var/www/html
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: <senha_do_banco>
    volumes:
      - db_data:/var/lib/mysql
volumes:
  wordpress_data:
  db_data:
" > docker-compose.yml

# Iniciar containers
docker-compose up -d

# 7. Demonstração da Aplicação Wordpress

# Acesse a URL fornecida pelo Load Balancer para ver a tela de login do Wordpress.
# O Wordpress deve estar rodando na porta 8080.
