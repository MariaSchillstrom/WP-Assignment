

## 5.5 RDS – databas för WordPress (MANUELLT → därefter IaC)

### 5.5.1 Provisionering via CloudFormation

Jag satte först upp EFS manuellt, för att sedan exportera/återskapa via IaC Generator.

**YAML-mall (rds.yaml):**

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Parameters:
  VpcSecurityGroupIds:
    Type: List<AWS::EC2::SecurityGroup::Id>
    Description: "RDS SG(s)"

  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: "Minst två subnät i olika AZ"

  DBInstanceClass:
    Type: String
    Default: "db.t4g.micro"

  AllocatedStorage:
    Type: Number
    Default: 20

  MaxAllocatedStorage:
    Type: Number
    Default: 100

  DBName:
    Type: String
    Default: "wordpress"

  MasterUsername:
    Type: String
    Default: "admin"

  DBEngineVersion:
    Type: String
    Default: "8.0.42"

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: "Subnets for RDS"
      SubnetIds: !Ref SubnetIds

  MyDB:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    UpdateReplacePolicy: Delete
    Properties:
      Engine: mysql
      EngineVersion: !Ref DBEngineVersion
      DBInstanceClass: !Ref DBInstanceClass
      AllocatedStorage: !Ref AllocatedStorage
      MaxAllocatedStorage: !Ref MaxAllocatedStorage
      StorageType: gp2
      StorageEncrypted: true
      MultiAZ: false
      PubliclyAccessible: true
      AutoMinorVersionUpgrade: true
      CopyTagsToSnapshot: true
      BackupRetentionPeriod: 0
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups: !Ref VpcSecurityGroupIds
      DBName: !Ref DBName
      MasterUsername: !Ref MasterUsername
      ManageMasterUserPassword: true

Outputs:
  DBEndpointAddress:
    Value: !GetAtt MyDB.Endpoint.Address
  DBSecretArn:
    Value: !GetAtt MyDB.MasterUserSecret.SecretArn
```

**Parameterfil (rds-params.json):**

```json
[
  {"ParameterKey":"VpcSecurityGroupIds","ParameterValue":"sg-072c202d2c475594e"},
  {"ParameterKey":"SubnetIds","ParameterValue":"subnet-09d87b7e1eda77420,subnet-0ac8592636998d30c,subnet-04d9a77bb77b5b320"},
  {"ParameterKey":"DBInstanceClass","ParameterValue":"db.t4g.micro"},
  {"ParameterKey":"AllocatedStorage","ParameterValue":"20"},
  {"ParameterKey":"MaxAllocatedStorage","ParameterValue":"100"},
  {"ParameterKey":"DBName","ParameterValue":"wordpress"},
  {"ParameterKey":"MasterUsername","ParameterValue":"admin"}
]
```

**Kommando för att skapa stacken:**

```bash
aws cloudformation create-stack \
  --stack-name rds-wp \
  --template-body file://rds.yaml \
  --parameters file://rds-params.json \
  --capabilities CAPABILITY_NAMED_IAM
```

### 5.5.2 Problem & lösning – Security Groups

När jag satte upp RDS fungerade inte anslutningen till MySQL Workbench direkt, trots att databasen hade en Security Group kopplad.

* **Orsak:** Regeln i SG var en SG→SG-regel (släpper bara in trafik från andra resurser i AWS). Min dator ansluter via publik IP och blockerades därför.
* **Lösning:** Jag lade till en inbound-regel på port 3306 för min publika IP/32. Därefter fungerade anslutningen.
* **Framåt:** Detta kan lösas direkt i CloudFormation med en parameter för administratörens IP-adress, i stället för manuell ändring.

### 5.5.3 Verifiering

* **Secrets Manager:** Lösen lagras automatiskt där p.g.a. `ManageMasterUserPassword = true`.
* **Workbench:** Anslut via RDS-endpoint, port 3306, användarnamn `admin`, lösenord från Secrets Manager.

 Resultat: En fungerande RDS MySQL-instans för WordPress.

[Workbench](Images/Workbench.png)

---

## 5.6 EFS för media (MANUELLT → därefter IaC)

Jag satte först upp EFS manuellt, för att sedan exportera/återskapa via IaC Generator.

```bash
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls <efs-id>:/ /mnt/efs
sudo mkdir -p /mnt/efs/wp-uploads
sudo rsync -a /var/www/html/wordpress/wp-content/uploads/ /mnt/efs/wp-uploads/
sudo mv /var/www/html/wordpress/wp-content/uploads /var/www/html/wordpress/wp-content/uploads.bak
sudo ln -s /mnt/efs/wp-uploads /var/www/html/wordpress/wp-content/uploads
# fstab
 echo "<efs-id>:/ /mnt/efs efs tls,_netdev 0 0" | sudo tee -a /etc/fstab
```

**Notis:** *EFS sattes först upp manuellt. Därefter exporterades via IaC Generator till en CloudFormation-mall.*

**Utkast (EFS1.yaml):**

AWSTemplateFormatVersion: "2010-09-09"

```
Parameters:
  VPC:
    Type: AWS::EC2::VPC::Id
  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
  AlbSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
  WebSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
  LatestAmiId:
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"

Resources:
  EC2TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VPC
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckProtocol: HTTP
      HealthCheckPath: /healthz.html
      Matcher: { HttpCode: "200-399" }

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Type: application
      Subnets: !Ref Subnets
      SecurityGroups: [ !Ref AlbSecurityGroupId ]

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref EC2TargetGroup


```


---

## 6. Drift, uppdatering & rollback (kursnivå)

* **Ny version:** ändra på fristående EC2 → skapa **ny AMI** → uppdatera **AmiId** → **Instance Refresh**.
* **Rollback:** peka tillbaka `AmiId` till föregående AMI → ny **Instance Refresh**.

---

## 7. Felsökning (kort)

* **504 från ALB:** kontrollera RDS-SG (3306 från Web-SG), TG-timeout, HealthCheckPath.
* **500 lokalt:** PHP-FPM saknas/fel socket-ägare (se 5.4).
* **404 på `info.php`:** filen inte i `/var/www/html`.
* **Git Bash path-conversion:** använd PowerShell eller `MSYS_NO_PATHCONV=1`.

---

## 8. Reflektion & fortsatt arbete

* Lägg till **HTTPS/ACM**, **WAF**, autoscaling-policys, **backup/restore-rutiner** för RDS/EFS, och **CI/CD** för AMI-byggen när kursmomenten täcker detta.

---

**Snabbverifiering:**

```bash
nc -zv <rds-endpoint> 3306
nc -zv <efs-mount-target-ip> 2049
```








