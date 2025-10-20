# 🐳 Manual de Deploy ECS Fargate (VPC Default)

## 🧠 O que você aprendeu
Você liberou **saída (egress)** no Security Group. Isso permitiu que a task ECS conseguisse se conectar à Internet (porta 443) e baixar a imagem do ECR.  
O erro `i/o timeout` ocorre quando a task não tem acesso à Internet ou saída TCP.  

---

## 🚀 Passo a passo: Deploy ECS Fargate Simples (usando VPC Default)

### 📋 Pré-requisitos
1. AWS CLI configurado com permissões suficientes.
2. Role de execução para ECS Tasks (`ecsTaskExecutionRole`).
3. ECR com a imagem desejada (pode usar NGINX público).

---

## 🧱 Parte A — Deploy simples (sem ALB)

### 1️⃣ Criar Cluster
```bash
aws ecs create-cluster --cluster-name demo-fargate --region us-east-1
```

### 2️⃣ Criar Security Group
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text)

SG_ID=$(aws ec2 create-security-group   --group-name sg-demo-ecs   --description "SG ECS demo (egress liberado; inbound 80)"   --vpc-id $VPC_ID   --query GroupId --output text)

aws ec2 authorize-security-group-egress --group-id $SG_ID --protocol -1 --port all --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### 3️⃣ Criar Task Definition (exemplo com NGINX)
```json
{
  "family": "task-demo",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "public.ecr.aws/nginx/nginx:latest",
      "essential": true,
      "portMappings": [{ "containerPort": 80, "hostPort": 80, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/demo-ecs",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

Registrar:
```bash
aws ecs register-task-definition --cli-input-json file://task-demo.json
```

### 4️⃣ Criar Service
```bash
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true   --query "Subnets[].SubnetId" --output text)

aws ecs create-service   --cluster demo-fargate   --service-name svc-demo   --task-definition task-demo   --desired-count 1   --launch-type FARGATE   --network-configuration "awsvpcConfiguration={subnets=[$(echo $SUBNETS | sed 's/ /,/g')],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"   --region us-east-1
```

### 5️⃣ Testar
Pegue o **Public IP** da task no console ECS e acesse `http://<public-ip>/`

---

## 🧱 Parte B — Deploy com ALB (produção)

### 1️⃣ Criar Target Group
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].VpcId" --output text)

TG_ARN=$(aws elbv2 create-target-group   --name tg-demo-3000   --protocol HTTP   --port 3000   --target-type ip   --vpc-id $VPC_ID   --query "TargetGroups[0].TargetGroupArn" --output text)
```

### 2️⃣ Criar ALB
```bash
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true   --query "Subnets[].SubnetId" --output text)

ALB_SG=$(aws ec2 create-security-group --group-name sg-alb-demo   --description "ALB demo" --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

ALB_ARN=$(aws elbv2 create-load-balancer   --name alb-demo   --type application   --scheme internet-facing   --security-groups $ALB_SG   --subnets $(echo $SUBNETS)   --query "LoadBalancers[0].LoadBalancerArn" --output text)
```

### 3️⃣ Criar Listener (HTTPS → TG)
```bash
aws elbv2 create-listener   --load-balancer-arn $ALB_ARN   --protocol HTTPS   --port 443   --certificates CertificateArn=<CERT_ARN>   --ssl-policy ELBSecurityPolicy-TLS13-1-2-Res-2021-06   --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### 4️⃣ Criar SG da Task
```bash
APP_SG=$(aws ec2 create-security-group   --group-name sg-ecs-app   --description "ECS app behind ALB"   --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-egress --group-id $APP_SG --protocol -1 --port all --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 3000 --source-group $ALB_SG
```

### 5️⃣ Criar Service com ALB
```bash
aws ecs create-service   --cluster demo-fargate   --service-name svc-demo-alb   --task-definition task-demo-alb   --desired-count 1   --launch-type FARGATE   --load-balancers "targetGroupArn=$TG_ARN,containerName=app,containerPort=3000"   --network-configuration "awsvpcConfiguration={subnets=[$(echo $SUBNETS | sed 's/ /,/g')],securityGroups=[$APP_SG],assignPublicIp=ENABLED}"   --region us-east-1
```

---

## ✅ Resumo rápido
| Item | O que fazer | Motivo |
|------|--------------|--------|
| **Egress no SG** | Liberar `All traffic → 0.0.0.0/0` | Permitir saída pro ECR |
| **assignPublicIp** | Habilitar | Dar acesso à Internet |
| **Portas TG ↔ Container** | Iguais | Registrar corretamente |
| **Logs CloudWatch** | Verificar `/ecs/demo-ecs` | Diagnóstico e debugging |

---

💡 Depois disso, o site sobe via `http://<PublicIP>` ou `https://<ALB-DNS>`.  
