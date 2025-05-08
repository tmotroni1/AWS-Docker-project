# Projeto DevSecOps - Infraestrutura na AWS com Docker (Simulação Terminal)

# 1. Criar VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Criar subnets públicas
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.2.0/24 --availability-zone us-east-1b

# Criar subnets privadas
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.3.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.4.0/24 --availability-zone us-east-1b

# Criar e associar Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-12345678 --internet-gateway-id igw-12345678

# Criar NAT Gateway em subnet pública
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id subnet-10.0.1.0 --allocation-id eipalloc-12345678

# 2. Criar RDS MySQL
aws rds create-db-instance \
  --db-instance-identifier devsecops-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-db \
  --db-subnet-group-name my-private-subnet-group \
  --master-username admin \
  --master-user-password secret123

# 3. Criar EFS
aws efs create-file-system --creation-token devsecops-efs

# Criar mount targets para cada subnet privada
aws efs create-mount-target --file-system-id fs-1234 --subnet-id subnet-10.0.3.0 --security-groups sg-efs
aws efs create-mount-target --file-system-id fs-1234 --subnet-id subnet-10.0.4.0 --security-groups sg-efs

# 4. Criar Launch Template com Docker e montagem do EFS
echo "#!/bin/bash
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo mkdir /mnt/efs
sudo apt install -y nfs-common
sudo mount -t nfs4 -o nfsvers=4.1 fs-1234.efs.us-east-1.amazonaws.com:/ /mnt/efs
docker run -d -p 80:80 my-app-image
" > user_data.sh

aws ec2 create-launch-template \
  --launch-template-name devsecops-template \
  --version-description v1 \
  --launch-template-data file://template.json

# 5. Criar Application Load Balancer
aws elbv2 create-load-balancer --name devsecops-alb \
  --subnets subnet-10.0.1.0 subnet-10.0.2.0 \
  --security-groups sg-alb \
  --scheme internet-facing \
  --type application

aws elbv2 create-target-group \
  --name devsecops-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-12345678

aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-12345678

aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# 6. Configurar Security Groups
# ALB: permite entrada 0.0.0.0/0 porta 80
# EC2: permite entrada apenas do ALB porta 80
# RDS: entrada só da EC2 porta 3306
# EFS: entrada só da EC2 porta 2049

# 7. Acessar Load Balancer
curl http://devsecops-alb-123456.elb.amazonaws.com

# Esperado:
# Aplicação DevSecOps funcionando corretamente!
