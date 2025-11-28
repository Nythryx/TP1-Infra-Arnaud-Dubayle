# TP2-Infra-Arnaud-Dubayle

# Script Terraform pour créer l'infrastructure globale 
### Script à copier/coller dans le cloudshell de AWS

- Console AWS en anglais (US)
- Région : ca-central-1 (Canada)
- x=10
- Tous les noms utilisant TRI deviennent ARN_

```bash
#!/bin/bash
set -e

echo "=== VARIABLES ==="

TRI="ARN"
X=10
VPC1_CIDR="10.${X}.0.0/16"
VPC1_PUBLIC_CIDR="10.${X}.1.0/24"
VPC1_PRIVATE_CIDR="10.${X}.2.0/24"

VPC2_CIDR="10.$((100+X)).0.0/16"
VPC2_PUBLIC_CIDR="10.$((100+X)).1.0/24"
VPC2_PRIVATE_CIDR="10.$((100+X)).2.0/24"

AMI_HTTPD="ami-0c02fb55956c7d316"   # Amazon Linux 2023 + HTTPD (exemple)
AMI_BASIC="ami-0c02fb55956c7d316"

KEYNAME="${TRI}_Key"

echo "=== CREATION KEYPAIR ==="
aws ec2 create-key-pair --key-name "$KEYNAME" \
    --query "KeyMaterial" --output text > ${KEYNAME}.pem
chmod 400 ${KEYNAME}.pem

echo "=== CREATION VPC 1 ==="
VPC1=$(aws ec2 create-vpc --cidr-block $VPC1_CIDR --query "Vpc.VpcId" --output text)
aws ec2 create-tags --resources $VPC1 --tags Key=Name,Value=${TRI}_VPC1

echo "=== CREATION VPC 2 ==="
VPC2=$(aws ec2 create-vpc --cidr-block $VPC2_CIDR --query "Vpc.VpcId" --output text)
aws ec2 create-tags --resources $VPC2 --tags Key=Name,Value=${TRI}_VPC2


echo "=== CREATION SUBNETS POUR VPC1 ==="
PUB1=$(aws ec2 create-subnet --vpc-id $VPC1 --cidr-block $VPC1_PUBLIC_CIDR --query "Subnet.SubnetId" --output text)
aws ec2 create-tags --resources $PUB1 --tags Key=Name,Value=${TRI}_VPC1_Public

PRIV1=$(aws ec2 create-subnet --vpc-id $VPC1 --cidr-block $VPC1_PRIVATE_CIDR --query "Subnet.SubnetId" --output text)
aws ec2 create-tags --resources $PRIV1 --tags Key=Name,Value=${TRI}_VPC1_Private


echo "=== CREATION SUBNETS POUR VPC2 ==="
PUB2=$(aws ec2 create-subnet --vpc-id $VPC2 --cidr-block $VPC2_PUBLIC_CIDR --query "Subnet.SubnetId" --output text)
aws ec2 create-tags --resources $PUB2 --tags Key=Name,Value=${TRI}_VPC2_Public

PRIV2=$(aws ec2 create-subnet --vpc-id $VPC2 --cidr-block $VPC2_PRIVATE_CIDR --query "Subnet.SubnetId" --output text)
aws ec2 create-tags --resources $PRIV2 --tags Key=Name,Value=${TRI}_VPC2_Private


echo "=== INTERNET GATEWAYS ==="
IGW1=$(aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW1 --vpc-id $VPC1
aws ec2 create-tags --resources $IGW1 --tags Key=Name,Value=${TRI}_IGW1

IGW2=$(aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW2 --vpc-id $VPC2
aws ec2 create-tags --resources $IGW2 --tags Key=Name,Value=${TRI}_IGW2


echo "=== ROUTE TABLES ==="

# VPC1
RTB1=$(aws ec2 create-route-table --vpc-id $VPC1 --query "RouteTable.RouteTableId" --output text)
aws ec2 create-tags --resources $RTB1 --tags Key=Name,Value=${TRI}_RT_Public1
aws ec2 associate-route-table --subnet-id $PUB1 --route-table-id $RTB1
aws ec2 create-route --route-table-id $RTB1 --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW1

# VPC2
RTB2=$(aws ec2 create-route-table --vpc-id $VPC2 --query "RouteTable.RouteTableId" --output text)
aws ec2 create-tags --resources $RTB2 --tags Key=Name,Value=${TRI}_RT_Public2
aws ec2 associate-route-table --subnet-id $PUB2 --route-table-id $RTB2
aws ec2 create-route --route-table-id $RTB2 --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW2


echo "=== SECURITY GROUPS ==="

# Bastions = SSH public
SG_BASTION1=$(aws ec2 create-security-group --group-name ${TRI}_SG_Bastion1 --description "Bastion1" --vpc-id $VPC1 --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_BASTION1 --protocol tcp --port 22 --cidr 0.0.0.0/0

SG_BASTION2=$(aws ec2 create-security-group --group-name ${TRI}_SG_Bastion2 --description "Bastion2" --vpc-id $VPC2 --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_BASTION2 --protocol tcp --port 22 --cidr 0.0.0.0/0

# Private = HTTP ONLY (modifié après peering)
SG_PRIV1=$(aws ec2 create-security-group --group-name ${TRI}_SG_Priv1 --description "Priv1" --vpc-id $VPC1 --query "GroupId" --output text)
SG_PRIV2=$(aws ec2 create-security-group --group-name ${TRI}_SG_Priv2 --description "Priv2" --vpc-id $VPC2 --query "GroupId" --output text)


echo "=== INSTANCES ==="

echo "--- Bastion VPC1 ---"
BASTION1=$(aws ec2 run-instances \
    --image-id $AMI_BASIC \
    --instance-type t2.micro \
    --key-name $KEYNAME \
    --security-group-ids $SG_BASTION1 \
    --subnet-id $PUB1 \
    --associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TRI}_BastionVPC1}]" \
    --query "Instances[0].InstanceId" --output text)

echo "--- Bastion VPC2 ---"
BASTION2=$(aws ec2 run-instances \
    --image-id $AMI_BASIC \
    --instance-type t2.micro \
    --key-name $KEYNAME \
    --security-group-ids $SG_BASTION2 \
    --subnet-id $PUB2 \
    --associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TRI}_BastionVPC2}]" \
    --query "Instances[0].InstanceId" --output text)

echo "--- Private VPC1 ---"
PRIVINST1=$(aws ec2 run-instances \
    --image-id $AMI_HTTPD \
    --instance-type t2.micro \
    --key-name $KEYNAME \
    --security-group-ids $SG_PRIV1 \
    --subnet-id $PRIV1 \
    --no-associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TRI}_InstanceVPC1}]" \
    --query "Instances[0].InstanceId" --output text)

echo "--- Private VPC2 ---"
PRIVINST2=$(aws ec2 run-instances \
    --image-id $AMI_HTTPD \
    --instance-type t2.micro \
    --key-name $KEYNAME \
    --security-group-ids $SG_PRIV2 \
    --subnet-id $PRIV2 \
    --no-associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TRI}_InstanceVPC2}]" \
    --query "Instances[0].InstanceId" --output text)


echo "=== PEERING ==="
PEER=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $VPC1 --peer-vpc-id $VPC2 \
    --query "VpcPeeringConnection.VpcPeeringConnectionId" \
    --output text)

aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER

# ajout routes
aws ec2 create-route --route-table-id $RTB1 --destination-cidr-block $VPC2_CIDR --vpc-peering-connection-id $PEER
aws ec2 create-route --route-table-id $RTB2 --destination-cidr-block $VPC1_CIDR --vpc-peering-connection-id $PEER


echo "=== HTTP RULES ==="
aws ec2 authorize-security-group-ingress --group-id $SG_PRIV1 --protocol tcp --port 80 --cidr $VPC2_CIDR
aws ec2 authorize-security-group-ingress --group-id $SG_PRIV2 --protocol tcp --port 80 --cidr $VPC1_CIDR


echo "=== FIN ==="

echo "BASTION 1 PUBLIC IP:"
aws ec2 describe-instances --instance-ids $BASTION1 --query "Reservations[0].Instances[0].PublicIpAddress" --output text

echo "BASTION 2 PUBLIC IP:"
aws ec2 describe-instances --instance-ids $BASTION2 --query "Reservations[0].Instances[0].PublicIpAddress" --output text

echo "INSTANCE PRIVEE 1 PRIVATE IP:"
aws ec2 describe-instances --instance-ids $PRIVINST1 --query "Reservations[0].Instances[0].PrivateIpAddress" --output text

echo "INSTANCE PRIVEE 2 PRIVATE IP:"
aws ec2 describe-instances --instance-ids $PRIVINST2 --query "Reservations[0].Instances[0].PrivateIpAddress" --output text
```

