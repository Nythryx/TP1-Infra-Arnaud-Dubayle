# TP1-Infra-Arnaud-Dubayle

# Partie 1 : Création d'un Load Balancer (LB)

## 1 Créer un Target Group

1. Connexion à la console AWS.
2. EC2 → Target Groups
3. Cliquer sur Create target group
4. Paramètres :
   * **Target type** : `Instances`
   * **Name** : `ARD_TargetGroup`
   * **Protocol** : `HTTP`
   * **Port** : `80`
   * **VPC** : sélectionne le VPC où sont tes instances EC2
5. Cliquer sur **Next**
6. Cliquer sur Create target group

## 2 Créer un Load Balancer

1. Aller dans **EC2 → Load Balancers**
2. Cliquer sur **Create Load Balancer → Application Load Balancer**
3. Paramètres :

   * **Name** : `ARD_LoadBalancer`
   * **Scheme** : `internet-facing`
   * **IP address type** : `IPv4`
4. **Listeners** : laisser par défaut `HTTP 80`
5. **Availability Zones** : sélectionner au moins 2 subnets
6. Dans **Target Group**, choisr **Existing target group** et sélectionner `ARD_TargetGroup`
8. Cliquer sur **Create Load Balancer**

## 3 Créer un Security Group pour le LB

1. Aller dans **EC2 → Security Groups**
2. Cliquer sur **Create Security Group**
3. Paramètres :

   * **Name** : `ARD_SecurityGroup_LB`
   * **VPC** : même VPC que le LB
4. Dans **Inbound rules** :

   * Type : `HTTP`
   * Protocol : `TCP`
   * Port : `80`
   * Source : **My IP** (AWS détecte IP publique automatiquement)
5. Cliquer sur **Create security group**

## 4 Appliquer le Security Group sur le Load Balancer

1. Retourner dans **EC2 → Load Balancers**
2. Sélectionner `ARD_LoadBalancer`
3. Cliquer sur **Actions → Edit security groups**
4. Cocher `ARD_SecurityGroup_LB`
5. Sauvegarder


# Partie 2 : Création d’une Amazon Machine Image (AMI)

## 1 Lancer une instance EC2 de base
1.  Connectez-vous à la console AWS
2.  Aller dans **EC2 → Instances → Launch instances**
3.  Paramètres :   
    -   **AMI** : Amazon Linux 2  
    -   **Instance type** : t2.micro (ou selon besoin)  
    -   **Key Pair** : sélectionner une clé existante ou en créer une nouvelle pour SSH 
    -   **Network & Subnet** : choisir le VPC et subnet appropriés  
    -   **Security Group** : autoriser SSH et HTTP (port 22 et 80)   
4.  Lancer l’instance.
    

## 2 Installer un serveur web

1.  Se connecter à l’instance via SSH :
    ```bash
    ssh -i votre_cle.pem ec2-user@IP_de_l_instance
    ```
    
2.  Installer Apache HTTP Server :
    ```bash
    sudo yum update -y
    sudo yum install -y httpd
    sudo systemctl start httpd
    sudo systemctl enable httpd
    ```
    

## 3 Créer un script pour récupérer et afficher les métadonnées

1.  Création du script shell :
    ```bash
    sudo nano /var/www/html/metadata.sh
    ```

2.  Contenu du script :
    ```bash
    #!/bin/bash
    TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
      -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    curl -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/instance-id > /var/www/html/index.html
    ```
    
3.  Commande pour rendre le script exécutable :
    ```bash
    sudo chmod +x /var/www/html/metadata.sh
    ```
    

## 4 Automatiser l'exécution du script au démarrage

1.  Éditer le crontab root :
    ```bash
    sudo crontab -e
    ```
    
2.  Ajouter la ligne suivante :
    ```bash
    @reboot /var/www/html/metadata.sh
    ```
3.  Sauvegarder et quitter
    

## 5 Créer une AMI personnalisée

1.  Retourner dans **EC2 → Instances**
2.  Sélectionner l'instance configurée  
3.  Cliquer sur **Actions → Image and templates → Create image**   
4.  Paramètres :
    -   **Image name** : nom de l’AMI personnalisé
    -   **Image description** : description facultative
5.  Cliquer sur **Create Image**


# Partie 3 : Création de la première instance EC2 et configuration des security groups

## 1 Créer un Security Group

1. Aller dans **EC2 → Security Groups → Create Security Group**
2. Paramètres :
   * **Name** : `ARD_SecurityGroup_EC2`
   * **VPC** : choisir le VPC approprié

3. Ajouter les règles suivantes :
   * **SSH (port 22)** : autoriser l'accès depuis votre adresse IP publique ([https://ifconfig.me](https://ifconfig.me) pour connaître l’IP)
   * **HTTP (port 80)** : autoriser uniquement le trafic provenant du security group du load balancer

4. Ajouter votre instance créée précédemment à ce Security Group

## 2 Créer une instance EC2

1. Aller dans **EC2 → Instances → Launch Instances**

2. Paramètres :
   * **Name** : `ARD_Instance1`
   * **AMI** : choisir l’AMI personnalisée créée précédemment
   * **Instance type** : `t2.micro`
   * **Security Group** : sélectionner `ARD_SecurityGroup_EC2`

3. Lancer l’instance

## 3 Vérification de l’accès au serveur web

1. Utiliser l’adresse du Load Balancer dans un navigateur pour accéder au serveur web et aux métadonnées de l’instance EC2

💡 Astuce : si le serveur web n’est pas accessible via le Load Balancer, procéder par étapes :

* Accéder directement à l’instance via votre navigateur (en s’assurant que le port 80 est accessible depuis votre IP)
* Modifier les règles de sécurité pour permettre l’accès via le Load Balancer et tester à nouveau


# Partie 4 : Installation de l'AWS CLI et ajout d’une seconde instance

## 1 Installer et configurer l’AWS CLI sur un ordinateur

1.  Télécharger et installer l’AWS CLI en suivant la documentation officielle
    -   [https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
        
2.  Configurer l’AWS CLI avec les identifiants IAM fournis
    ```bash
    aws configure
    ```
    
3.  Renseigner les informations demandées :
    -   AWS Access Key ID
    -   AWS Secret Access Key
    -   Default region name
    -   Default output format
        

## 2 Créer une seconde instance EC2 avec l’AWS CLI

1.  Lancer une instance similaire à la première en utilisant l’AMI personnalisée
    
2.  Exemple de commande :
    ```bash
    aws ec2 run-instances \
      --image-id <ID_AMI_PERSONNALIEE> \
      --count 1 \
      --instance-type t2.micro \
      --security-group-ids <ID_ARN_SecurityGroup_EC2> \
      --subnet-id <ID_SUBNET> \
      --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=TRI_Instance2}]'
    ```
    
    3.  Noter l’Instance ID retourné par la commande
    

## 3 Ajouter cette instance au Target Group via l’AWS CLI

1.  Récupérer l’ARN du Target Group `ARN_TargetGroup`
    ```bash
    aws elbv2 describe-target-groups
    ```
    
2.  Ajouter la nouvelle instance au Target Group
    ```bash
    aws elbv2 register-targets \
      --target-group-arn <ARN_TARGET_GROUP> \
      --targets Id=<INSTANCE_ID_INSTANCE2>,Port=80
    ```
    

## 4 Vérification via le Load Balancer

1.  Ouvrir l’adresse du Load Balancer dans un navigateur
2.  Rafraîchir plusieurs fois la page pour vérifier que le Load Balancer distribue bien le trafic entre les deux instances
3.  Si nécessaire, vider le cache du navigateur ou utiliser le mode navigation privée

