# ECS CloudFormation Deployment Guide

This repository contains AWS CloudFormation templates and GitHub Actions workflows for deploying the SacuNxt ECS infrastructure with **AWS-managed services** for Redis (ElastiCache) and Kafka (Amazon MSK).

## 🏗️ Architecture Overview

The deployment creates a complete microservices infrastructure using AWS CloudFormation:

### Infrastructure Components

- **VPC & Networking**: Public and private subnets across 2 AZs with NAT Gateway
- **RDS PostgreSQL**: AWS-managed relational database (Multi-AZ capable)
- **ElastiCache Redis**: AWS-managed in-memory cache (Multi-node cluster capable)
- **Amazon MSK**: AWS-managed Apache Kafka cluster (Multi-broker setup)
- **ECS Fargate Cluster**: Serverless container orchestration
- **CloudWatch Logs**: Centralized logging for all services
- **Service Discovery**: AWS CloudMap for service-to-service communication
- **IAM Roles**: Least-privilege security for ECS tasks with auto-creation option

### Microservices Deployed

1. **API Service** (Port 8000) - Main API gateway with service discovery
2. **Scheduler Service** (Port 8002) - Job scheduling and orchestration
3. **Config Service** (Port 8001) - Configuration management
4. **Collector Service** (Port 8003) - Data collection and ingestion
5. **Normalizer Service** (Port 8004) - Data normalization
6. **Publisher Service** (Port 8005) - Data publishing

### Key Features

- **Automated IAM Role Management**: Option to create ECS task execution role automatically
- **Cross-Account ECR Replication**: Support for cross-account image replication
- **Enhanced Logging**: Stack-specific log groups with proper naming
- **Service Discovery Integration**: AWS CloudMap for internal service communication
- **Infrastructure as Code**: Complete CloudFormation-based deployment

## 📋 Prerequisites

### Required AWS Services
- AWS Account with appropriate permissions
- ECR Repository for container images
- AWS CLI configured

### Required Tools
- Git
- AWS CLI v2

### Required Repository Secrets
Configure these secrets in your GitHub repository settings:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `AWS_REGION` | AWS region |
| `ORG_READ_TOKEN` | GitHub token for team verification |
| `DB_USERNAME` | Database master username |
| `DB_PASSWORD` | Database master password |

## 🚀 Deployment Methods

### Method 1: GitHub Actions (Recommended)

#### Step 1: Configure Repository Secrets
1. Go to your repository → Settings → Secrets and variables → Actions
2. Add all required secrets listed above

#### Step 2: Trigger Deployment
1. Go to Actions tab in your repository
2. Select "CloudFormation ECS Deployment" workflow
3. Click "Run workflow"
4. Enter deployment confirmation: `deploy`
5. Select action:
   - `create-or-update` - Deploy or update infrastructure
   - `delete` - Remove all resources
6. Click "Run workflow"

#### Step 3: Monitor Progress
- Watch the workflow execution in real-time
- Stacks are deployed in dependency order:
  1. VPC and Networking
  2. RDS PostgreSQL
  3. ElastiCache Redis
  4. Amazon MSK Kafka
  5. ECS Cluster and Services
- Check stack outputs in the summary

### Method 2: Manual AWS CLI Deployment

#### Step 1: Clone Repository
```bash
git clone <repository-url>
cd ecs_cloudformation_deploy
```

#### Step 2: Configure AWS Credentials
```bash
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-2
```

#### Step 3: Deploy Stacks in Order

**1. Deploy VPC and Networking**
```bash
aws cloudformation deploy \
  --template-file templates/01-vpc-networking.yaml \
  --stack-name sacunxt-vpc-networking \
  --parameter-overrides ProjectName=sacunxt \
  --capabilities CAPABILITY_NAMED_IAM
```

**2. Deploy RDS PostgreSQL**
```bash
aws cloudformation deploy \
  --template-file templates/04-rds-postgresql.yaml \
  --stack-name sacunxt-rds-postgresql \
  --parameter-overrides \
    ProjectName=sacunxt \
    DBUsername=sacunxt \
    DBPassword=your_secure_password \
  --capabilities CAPABILITY_NAMED_IAM
```

**3. Deploy ElastiCache Redis**
```bash
aws cloudformation deploy \
  --template-file templates/02-elasticache-redis.yaml \
  --stack-name sacunxt-elasticache-redis \
  --parameter-overrides ProjectName=sacunxt \
  --capabilities CAPABILITY_NAMED_IAM
```

**4. Deploy Amazon MSK Kafka**
```bash
aws cloudformation deploy \
  --template-file templates/03-msk-kafka.yaml \
  --stack-name sacunxt-msk-kafka \
  --parameter-overrides ProjectName=sacunxt \
  --capabilities CAPABILITY_NAMED_IAM
```

**5. Wait for MSK Cluster to be Active**
```bash
MSK_ARN=$(aws cloudformation describe-stacks \
  --stack-name sacunxt-msk-kafka \
  --query "Stacks[0].Outputs[?OutputKey=='MSKClusterArn'].OutputValue" \
  --output text)

aws kafka wait cluster-active --cluster-arn $MSK_ARN
```

**6. Deploy ECS Cluster and Services**
```bash
aws cloudformation deploy \
  --template-file templates/05-ecs-cluster-services.yaml \
  --stack-name sacunxt-ecs-services \
  --parameter-overrides \
    ProjectName=sacunxt \
    DBUsername=sacunxt \
    DBPassword=your_secure_password \
  --capabilities CAPABILITY_NAMED_IAM
```

## ⚙️ Configuration

### CloudFormation Parameters

Each template accepts parameters for customization:

#### VPC Template (01-vpc-networking.yaml)
| Parameter | Default | Description |
|-----------|---------|-------------|
| `ProjectName` | `sacunxt` | Project name prefix |
| `VpcCIDR` | `10.0.0.0/16` | VPC CIDR block |
| `PublicSubnet1CIDR` | `10.0.1.0/24` | Public subnet 1 |
| `PublicSubnet2CIDR` | `10.0.2.0/24` | Public subnet 2 |
| `PrivateSubnet1CIDR` | `10.0.3.0/24` | Private subnet 1 |
| `PrivateSubnet2CIDR` | `10.0.4.0/24` | Private subnet 2 |

#### ElastiCache Template (02-elasticache-redis.yaml)
| Parameter | Default | Description |
|-----------|---------|-------------|
| `RedisNodeType` | `cache.t3.micro` | Instance type |
| `RedisEngineVersion` | `7.0` | Redis version |
| `NumCacheNodes` | `1` | Number of nodes (1=single, 2+=cluster) |

#### MSK Template (03-msk-kafka.yaml)
| Parameter | Default | Description |
|-----------|---------|-------------|
| `KafkaVersion` | `3.5.1` | Kafka version |
| `BrokerInstanceType` | `kafka.t3.small` | Broker instance type |
| `NumberOfBrokerNodes` | `2` | Number of brokers |
| `VolumeSize` | `100` | EBS volume size (GB) |

#### RDS Template (04-rds-postgresql.yaml)
| Parameter | Default | Description |
|-----------|---------|-------------|
| `DBInstanceClass` | `db.t3.micro` | Instance class |
| `DBUsername` | `sacunxt` | Master username |
| `DBPassword` | - | Master password (required) |
| `DBName` | `api_service` | Initial database |
| `AllocatedStorage` | `20` | Storage in GB |
| `BackupRetentionPeriod` | `7` | Backup retention days |

#### ECS Template (05-ecs-cluster-services.yaml)
| Parameter | Default | Description |
|-----------|---------|-------------|
| `CreateTaskExecutionRole` | `true` | Auto-create ECS task execution role |
| `TaskExecutionRoleArn` | `''` | Existing role ARN when auto-creation disabled |
| `ECRRepositoryURL` | `<your-ecr-url>` | ECR URL |
| `APIServiceVersion` | `2.3.4` | API service version |
| `SchedulerServiceVersion` | `2.3.4` | Scheduler version |
| `ConfigServiceVersion` | `2.3.2` | Config version |
| `CollectorServiceVersion` | `2.3.13` | Collector version |
| `NormalizerServiceVersion` | `2.3.7` | Normalizer version |
| `PublisherServiceVersion` | `2.3.10` | Publisher version |

### IAM Role Management

The ECS template now supports two modes:

#### Mode 1: Automatic Role Creation (Default)
```bash
aws cloudformation deploy \
  --template-file templates/05-ecs-cluster-services.yaml \
  --stack-name sacunxt-ecs-services \
  --parameter-overrides \
    ProjectName=sacunxt \
    CreateTaskExecutionRole=true \
    DBUsername=sacunxt \
    DBPassword=your_secure_password
```

#### Mode 2: Use Existing Role
```bash
aws cloudformation deploy \
  --template-file templates/05-ecs-cluster-services.yaml \
  --stack-name sacunxt-ecs-services \
  --parameter-overrides \
    ProjectName=sacunxt \
    CreateTaskExecutionRole=false \
    TaskExecutionRoleArn=arn:aws:iam::ACCOUNT:role/your-existing-role \
    DBUsername=sacunxt \
    DBPassword=your_secure_password
```

### Updating Service Versions

Modify parameters during deployment:
```bash
aws cloudformation deploy \
  --template-file templates/05-ecs-cluster-services.yaml \
  --stack-name sacunxt-ecs-services \
  --parameter-overrides \
    APIServiceVersion=2.3.5 \
    SchedulerServiceVersion=2.3.5
```

### Scaling Services

Edit the task definition `DesiredCount` in the template or use AWS Console/CLI:
```bash
aws ecs update-service \
  --cluster sacunxt-cluster \
  --service api-service \
  --desired-count 3
```

## 🔍 Service Endpoints

After deployment, services are accessible via:

| Service | Endpoint | Port |
|---------|----------|------|
| API Service | `api-service.sacunxt.local` | 8000 |
| PostgreSQL | From RDS stack output | 5432 |
| Redis | From ElastiCache stack output | 6379 |
| Kafka | From MSK stack output | 9092 |

### Getting Endpoints

```bash
# PostgreSQL
aws cloudformation describe-stacks \
  --stack-name sacunxt-rds-postgresql \
  --query "Stacks[0].Outputs[?OutputKey=='DBConnectionString'].OutputValue" \
  --output text

# Redis
aws cloudformation describe-stacks \
  --stack-name sacunxt-elasticache-redis \
  --query "Stacks[0].Outputs[?OutputKey=='RedisConnectionString'].OutputValue" \
  --output text

# Kafka
aws cloudformation describe-stacks \
  --stack-name sacunxt-msk-kafka \
  --query "Stacks[0].Outputs[?OutputKey=='BootstrapBrokersPlaintext'].OutputValue" \
  --output text

# ECS Cluster Name
aws cloudformation describe-stacks \
  --stack-name sacunxt-ecs-services \
  --query "Stacks[0].Outputs[?OutputKey=='ECSClusterName'].OutputValue" \
  --output text
```

## 🛠️ Troubleshooting

### Common Issues

#### 1. Stack Creation Failed
```
❌ CREATE_FAILED
```
**Solution**: 
- Check CloudFormation events for specific error
- Verify all dependencies are deployed first
- Check AWS service quotas

#### 2. MSK Cluster Takes Long Time
```
⏳ Cluster creation in progress...
```
**Solution**: MSK cluster creation takes 15-30 minutes. This is normal.

#### 3. Service Not Starting
```
❌ ECS service tasks failing
```
**Solution**:
- Check CloudWatch logs for container errors
- Verify environment variables are correct
- Check security group rules

#### 4. Database Connection Failed
```
❌ Unable to connect to database
```
**Solution**:
- Verify RDS security group allows ECS tasks
- Check database credentials
- Ensure RDS is in AVAILABLE state

### Debugging Commands

#### Check Stack Status
```bash
aws cloudformation describe-stacks --stack-name sacunxt-vpc-networking
```

#### View Stack Events
```bash
aws cloudformation describe-stack-events --stack-name sacunxt-ecs-services
```

#### Check ECS Service Status
```bash
aws ecs describe-services \
  --cluster sacunxt-cluster \
  --services api-service
```

#### View Container Logs
```bash
aws logs tail /ecs/sacunxt/api-service --follow
```

#### Check MSK Cluster Status
```bash
aws kafka describe-cluster --cluster-arn <cluster-arn>
```

## 🔄 Updates and Maintenance

### Updating Infrastructure

Simply re-run the deployment with updated parameters:
```bash
aws cloudformation deploy \
  --template-file templates/05-ecs-cluster-services.yaml \
  --stack-name sacunxt-ecs-services \
  --parameter-overrides APIServiceVersion=2.3.5
```

### Rolling Updates

ECS services automatically perform rolling updates when task definitions change.

## 🧹 Cleanup

### Remove All Resources

**Via AWS CLI:**
```bash
# Delete in reverse order
aws cloudformation delete-stack --stack-name sacunxt-ecs-services
aws cloudformation wait stack-delete-complete --stack-name sacunxt-ecs-services

aws cloudformation delete-stack --stack-name sacunxt-msk-kafka
aws cloudformation wait stack-delete-complete --stack-name sacunxt-msk-kafka

aws cloudformation delete-stack --stack-name sacunxt-elasticache-redis
aws cloudformation wait stack-delete-complete --stack-name sacunxt-elasticache-redis

aws cloudformation delete-stack --stack-name sacunxt-rds-postgresql
aws cloudformation wait stack-delete-complete --stack-name sacunxt-rds-postgresql

aws cloudformation delete-stack --stack-name sacunxt-vpc-networking
aws cloudformation wait stack-delete-complete --stack-name sacunxt-vpc-networking
```

## 📊 Monitoring

### CloudWatch Metrics

All services publish metrics to CloudWatch:
- **ECS**: CPU, Memory, Task counts
- **RDS**: Database connections, CPU, storage
- **ElastiCache**: Cache hits/misses, CPU, memory
- **MSK**: Broker metrics, consumer lag

### CloudWatch Logs

Logs are organized by service:
- `/ecs/sacunxt/api-service`
- `/ecs/sacunxt/scheduler-service`
- `/ecs/sacunxt/config-service`
- `/ecs/sacunxt/collector-service`
- `/ecs/sacunxt/normalizer-service`
- `/ecs/sacunxt/publisher-service`
- `/aws/msk/sacunxt`

## 🔐 Security

### Network Security
- Private subnets for managed services (RDS, ElastiCache, MSK)
- Public subnets for ECS tasks with internet access
- Security groups with least-privilege rules
- NAT Gateway for outbound traffic from private subnets

### Data Encryption
- **RDS**: Encryption at rest enabled
- **ElastiCache**: Encryption at rest enabled
- **MSK**: Encryption in transit and at rest enabled
- **EBS**: Encrypted volumes for MSK brokers

### IAM Security
- Separate execution and task roles
- Least-privilege policies
- No hardcoded credentials
- Conditional IAM role creation with validation

### Secrets Management
- Database credentials passed via parameters
- Sensitive parameters marked with `NoEcho`
- Use AWS Secrets Manager for production

##  Cost Optimization

### Estimated Monthly Costs

| Service | Configuration | Estimated Cost |
|---------|--------------|----------------|
| RDS PostgreSQL | db.t3.micro | ~$15 |
| ElastiCache Redis | cache.t3.micro (1 node) | ~$12 |
| Amazon MSK | kafka.t3.small (2 brokers) | ~$150 |
| ECS Fargate | 6 services @ 0.25 vCPU | ~$43 |
| NAT Gateway | 1 gateway | ~$32 |
| **Total** | | **~$252/month** |

### Cost Reduction Tips
- Use smaller instance types for dev/test
- Reduce MSK broker count to 2 (minimum)
- Use Spot instances for non-critical services
- Enable auto-scaling for ECS services
- Delete unused resources

##  Migration from Terraform

This repository replaces the Terraform-based deployment with the following key differences:

| Feature | Terraform Version | CloudFormation Version |
|---------|------------------|------------------------|
| Redis | Self-hosted on ECS | AWS ElastiCache (Managed) |
| Kafka | Self-hosted on ECS | Amazon MSK (Managed) |
| PostgreSQL | Self-hosted on ECS | Amazon RDS (Managed) |
| Deployment Tool | Terraform | CloudFormation |
| State Management | Terraform state file | AWS CloudFormation |
| IAM Roles | Manual creation | Auto-creation option |
| Logging | Basic | Stack-specific log groups |
| Service Discovery | Basic | AWS CloudMap integration |

### Benefits of AWS-Managed Services
- ✅ Automated backups and snapshots
- ✅ Automated patching and updates
- ✅ Multi-AZ high availability
- ✅ Built-in monitoring and metrics
- ✅ Reduced operational overhead
- ✅ Better scalability options
- ✅ Enhanced security features

## 📋 Quick Reference

### Stack Dependencies
```
VPC Networking → RDS PostgreSQL → ElastiCache Redis → MSK Kafka → ECS Services
```

### Common Commands
```bash
# Check stack status
aws cloudformation list-stacks --query "StackSummaries[?StackStatus=='CREATE_COMPLETE']"

# Get all outputs
aws cloudformation describe-stacks --stack-name sacunxt-ecs-services --query "Stacks[0].Outputs"
```

---

**Last Updated**: March 2026  
**Version**: 2.0  
**Maintained by**: SacuNxt DevOps Team
