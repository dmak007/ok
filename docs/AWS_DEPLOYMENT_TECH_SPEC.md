# AWS Deployment Technical Specification

## Overview

This document outlines the technical approach for deploying the ok.py application to AWS with cost optimization as the primary goal, enabling scale-to-zero capabilities and on-demand spin-up/tear-down.

## Architecture Goals

1. **Cost Optimization**: Minimize costs for 1-2 user workload
2. **Scale to Zero**: Automatically scale down when not in use
3. **On-Demand**: Quick spin-up and tear-down capabilities
4. **Data Preservation**: Maintain data across tear-downs
5. **Infrastructure as Code**: Terraform-based with CDK migration path
6. **Containerized Tooling**: All CLI operations via Docker containers

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Cloud                            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    VPC (10.0.0.0/16)                   │ │
│  │                                                         │ │
│  │  ┌──────────────┐  ┌──────────────┐                  │ │
│  │  │ Public Subnet│  │ Public Subnet│                  │ │
│  │  │   (AZ-a)     │  │   (AZ-b)     │                  │ │
│  │  │              │  │              │                  │ │
│  │  │  ┌────────┐  │  │  ┌────────┐  │                  │ │
│  │  │  │  ALB   │  │  │  │  ALB   │  │                  │ │
│  │  │  └────────┘  │  │  └────────┘  │                  │ │
│  │  └──────────────┘  └──────────────┘                  │ │
│  │                                                         │ │
│  │  ┌──────────────┐  ┌──────────────┐                  │ │
│  │  │Private Subnet│  │Private Subnet│                  │ │
│  │  │   (AZ-a)     │  │   (AZ-b)     │                  │ │
│  │  │              │  │              │                  │ │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │                  │ │
│  │  │ │ECS Fargate│ │  │ │ECS Fargate│ │                  │ │
│  │  │ │           │ │  │ │           │ │                  │ │
│  │  │ │ ok-web    │ │  │ │ ok-worker │ │                  │ │
│  │  │ │ MySQL     │ │  │ │ Redis     │ │                  │ │
│  │  │ └──────────┘ │  │ └──────────┘ │                  │ │
│  │  └──────────────┘  └──────────────┘                  │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐       │
│  │     S3      │  │  S3 (MySQL/  │  │  ECR        │       │
│  │  (Storage)  │  │  Redis Data) │  │  (Images)   │       │
│  └─────────────┘  └──────────────┘  └─────────────┘       │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐                         │
│  │ CloudWatch  │  │  Secrets     │                         │
│  │  (Logs)     │  │  Manager     │                         │
│  └─────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Compute Layer (ECS Fargate)

**ECS Cluster**: Single cluster hosting all services

**Services**:
- **ok-web**: Flask application (0.25 vCPU, 512 MB RAM)
  - Desired count: 1 (can scale to 0)
  - Health check: `/healthz` endpoint
  - Auto-scaling based on CPU/Memory and request count
  
- **ok-worker**: RQ background worker (0.25 vCPU, 512 MB RAM)
  - Desired count: 1 (can scale to 0)
  - Processes background jobs from Redis queue
  
- **mysql**: MySQL 8.0 container (0.5 vCPU, 1 GB RAM)
  - Persistent storage via S3 (continuous sync)
  - Automated backups to S3 every 15 minutes
  - Single instance (sufficient for 1-2 users)
  - Uses s3fs-fuse or custom sync script
  
- **redis**: Redis 7.0 container (0.25 vCPU, 512 MB RAM)
  - In-memory storage with RDB snapshots to S3
  - Automated snapshots every hour
  - Single instance

**Cost Optimization**:
- Use Fargate Spot for non-critical workloads (70% cost savings)
- Scale to zero during inactivity (CloudWatch alarms + Lambda)
- Minimum task sizes (0.25 vCPU)

### 2. Networking

**VPC Configuration**:
- CIDR: 10.0.0.0/16
- 2 Public Subnets (10.0.1.0/24, 10.0.2.0/24)
- 2 Private Subnets (10.0.10.0/24, 10.0.11.0/24)
- NAT Gateway: Single NAT in one AZ (cost optimization)
- Internet Gateway for public access

**Load Balancer**:
- Application Load Balancer (ALB)
- HTTP/HTTPS listeners
- Target groups for ok-web service
- Health checks on `/healthz`

**Security Groups**:
- ALB SG: Allow 80/443 from 0.0.0.0/0
- ECS SG: Allow traffic from ALB SG
- MySQL SG: Allow 3306 from ECS SG
- Redis SG: Allow 6379 from ECS SG

### 3. Storage

**S3 Buckets**:
- `ok-storage-{env}`: Application file storage
- `ok-backups-{env}`: Database backups and snapshots
- `ok-mysql-data-{env}`: MySQL data backups (automated every 15 minutes)
- `ok-redis-data-{env}`: Redis RDB snapshots (automated hourly)
- Lifecycle policies: Move to Glacier after 30 days
- Versioning enabled for data preservation
- Intelligent-Tiering for cost optimization

**No EFS Required**: All persistent data stored in S3
- MySQL: Ephemeral container storage + continuous S3 sync
- Redis: In-memory + periodic RDB snapshots to S3
- Application files: Direct S3 integration (already supported)
- Cost savings: ~$5/month (no EFS charges)

### 4. Container Registry (ECR)

**Repositories**:
- `ok-web`: Flask application image
- `ok-worker`: Worker application image
- Lifecycle policy: Keep last 5 images, delete untagged after 7 days

### 5. Secrets Management

**AWS Secrets Manager**:
- Database credentials
- OAuth client secrets (Google, Microsoft)
- SendGrid API key
- Session secret key
- Storage credentials

### 6. Monitoring & Logging

**CloudWatch**:
- Log Groups: `/ecs/ok-web`, `/ecs/ok-worker`, `/ecs/mysql`, `/ecs/redis`
- Retention: 7 days (cost optimization)
- Alarms:
  - High CPU/Memory usage
  - No requests for 30 minutes (trigger scale-to-zero)
  - Health check failures

**Metrics**:
- ECS task count
- ALB request count
- Database connections
- Redis memory usage

### 7. Auto-Scaling & Scale-to-Zero

**Application Auto Scaling**:
- Target tracking: CPU 70%, Memory 80%
- Step scaling: Based on ALB request count
- Scale-in cooldown: 300 seconds
- Scale-out cooldown: 60 seconds

**Scale-to-Zero Lambda**:
- Triggered by CloudWatch alarm (no requests for 30 minutes)
- Triggers final MySQL/Redis backup to S3
- Sets ECS service desired count to 0
- Stops MySQL/Redis tasks (data preserved in S3)
- Estimated savings: ~$50-70/month when scaled to zero

**Scale-up Trigger**:
- ALB target group registration triggers Lambda
- Sets ECS service desired count to 1
- Starts MySQL/Redis tasks
- Warm-up time: ~2-3 minutes

## Infrastructure as Code Structure

```
infrastructure/
├── docker/
│   ├── Dockerfile.terraform      # Terraform CLI container
│   ├── Dockerfile.aws-cli        # AWS CLI container
│   └── Dockerfile.tools          # Combined tools container
├── terraform/
│   ├── modules/
│   │   ├── networking/           # VPC, subnets, security groups
│   │   ├── ecs/                  # ECS cluster, services, tasks
│   │   ├── storage/              # S3 buckets for data and backups
│   │   ├── database/             # MySQL ECS service with S3 sync
│   │   ├── cache/                # Redis ECS service with S3 snapshots
│   │   ├── monitoring/           # CloudWatch, alarms
│   │   ├── autoscaling/          # Auto-scaling policies
│   │   └── lambda/               # Scale-to-zero Lambda functions
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tfvars
│   │   ├── staging/
│   │   └── prod/
│   ├── backend.tf                # S3 backend configuration
│   └── versions.tf               # Provider versions
├── scripts/
│   ├── deploy.sh                 # Main deployment script
│   ├── destroy.sh                # Tear-down script
│   ├── scale-down.sh             # Manual scale-to-zero
│   ├── scale-up.sh               # Manual scale-up
│   ├── backup.sh                 # Manual backup trigger
│   ├── restore.sh                # Restore from backup
│   ├── build-images.sh           # Build and push Docker images
│   └── init-terraform.sh         # Initialize Terraform backend
├── lambda/
│   ├── scale-to-zero/
│   │   ├── index.py
│   │   └── requirements.txt
│   └── scale-up/
│       ├── index.py
│       └── requirements.txt
└── docs/
    ├── DEPLOYMENT.md             # Deployment guide
    ├── OPERATIONS.md             # Operations runbook
    └── COST_OPTIMIZATION.md      # Cost optimization strategies
```

## Deployment Workflow

### Prerequisites
- AWS SSO configured
- Docker installed
- Git repository access

### Initial Setup

```bash
# 1. Build CLI tools container
./scripts/build-tools.sh

# 2. Initialize Terraform backend (one-time)
./scripts/init-terraform.sh dev

# 3. Deploy infrastructure
./scripts/deploy.sh dev
```

### Daily Operations

```bash
# Scale down manually
./scripts/scale-down.sh dev

# Scale up manually
./scripts/scale-up.sh dev

# View logs
./scripts/logs.sh dev ok-web

# Backup database
./scripts/backup.sh dev

# Restore from backup
./scripts/restore.sh dev <backup-id>
```

### Tear-Down

```bash
# Destroy infrastructure (preserves data in S3/EFS snapshots)
./scripts/destroy.sh dev
```

## Cost Estimation

### Running Costs (24/7)
- ECS Fargate (4 tasks): ~$30-40/month
- ALB: ~$20/month
- NAT Gateway: ~$35/month
- S3: ~$5/month (storage + requests)
- CloudWatch Logs: ~$3/month
- **Total**: ~$93-103/month

### Optimized Costs (8 hours/day, 5 days/week)
- ECS Fargate: ~$10-15/month
- ALB: ~$20/month (always on)
- NAT Gateway: ~$10/month (scaled down)
- S3: ~$3/month (storage + requests)
- CloudWatch Logs: ~$2/month
- **Total**: ~$45-50/month

### Scale-to-Zero Costs (weekend/nights)
- ALB: ~$20/month (always on)
- S3: ~$2/month (storage only, minimal requests)
- CloudWatch: ~$1/month
- **Total**: ~$23/month

## Alternative: True Zero-Cost Architecture

For maximum cost savings when scaled to zero, an alternative architecture eliminates the always-on ALB:

### Architecture Changes

**Replace ALB with API Gateway + Lambda**:
- API Gateway HTTP API (pay-per-request)
- Lambda function to manage ALB lifecycle
- CloudFront (optional) for caching and SSL

### True Zero-Cost Components

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Cloud                            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              API Gateway HTTP API                      │ │
│  │         (Pay-per-request, ~$0 when idle)              │ │
│  └────────────────────────────────────────────────────────┘ │
│                            │                                 │
│                            ▼                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Lambda: ALB Lifecycle Manager                  │ │
│  │  - Creates ALB on first request                       │ │
│  │  - Returns "warming up" message                       │ │
│  │  - Polls for ALB ready state                          │ │
│  │  - Proxies requests once ready                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                            │                                 │
│                            ▼                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         ALB (Created on-demand)                        │ │
│  │         Destroyed after 30 min idle                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                            │                                 │
│                            ▼                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              ECS Fargate Services                      │ │
│  │         (Scaled up with ALB creation)                 │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Cost Breakdown (True Zero)

**When Scaled to Zero**:
- API Gateway: $0 (no requests)
- Lambda: $0 (no invocations)
- S3: ~$2/month (storage only, minimal requests)
- CloudWatch: ~$0.50/month (minimal logs)
- **Total**: ~$2-3/month

**When Active (8 hours/day, 5 days/week)**:
- API Gateway: ~$1/month (1M requests)
- Lambda: ~$2/month (ALB management)
- ALB: ~$7/month (160 hours)
- ECS Fargate: ~$10-15/month
- S3: ~$3/month (storage + requests)
- CloudWatch: ~$2/month
- **Total**: ~$25-30/month

### Trade-offs

| Aspect | Standard (ALB Always-On) | True Zero-Cost |
|--------|-------------------------|----------------|
| **Idle Cost** | ~$23/month | ~$2-3/month |
| **Active Cost (8hr/day)** | ~$45-50/month | ~$25-30/month |
| **Cold Start** | 2-3 minutes | 5-10 minutes |
| **Complexity** | Low | Medium |
| **First Request** | Fast | Returns "warming up" |
| **Subsequent Requests** | Fast | Fast (once warm) |
| **Best For** | Frequent access | Infrequent access |

### Implementation Details

**Lambda Function (ALB Lifecycle Manager)**:
```python
import boto3
import json
import time

ecs = boto3.client('ecs')
elbv2 = boto3.client('elbv2')
cloudformation = boto3.client('cloudformation')

def lambda_handler(event, context):
    # Check if ALB exists
    alb_arn = check_alb_exists()
    
    if not alb_arn:
        # Create ALB via CloudFormation
        stack_name = create_alb_stack()
        
        return {
            'statusCode': 202,
            'body': json.dumps({
                'message': 'System warming up, please wait 5-10 minutes',
                'status': 'creating',
                'retry_after': 300
            })
        }
    
    # Check if ECS services are running
    if not check_ecs_services_running():
        scale_up_ecs_services()
        
        return {
            'statusCode': 202,
            'body': json.dumps({
                'message': 'Services starting, please wait 2-3 minutes',
                'status': 'starting',
                'retry_after': 120
            })
        }
    
    # Proxy request to ALB
    return proxy_to_alb(event, alb_arn)
```

**Scale-Down Lambda**:
- Triggered by CloudWatch alarm (30 minutes no requests)
- Scales ECS services to 0
- Deletes ALB via CloudFormation
- Preserves all data on EFS

**API Gateway Configuration**:
- HTTP API (cheaper than REST API)
- Custom domain with Route53 (optional)
- Request timeout: 29 seconds (Lambda max)
- Throttling: 100 requests/second

### Migration Path

The infrastructure is designed to support both architectures:

1. **Start with Standard**: Deploy with ALB always-on for testing
2. **Monitor Usage**: Track actual usage patterns
3. **Switch to True Zero**: If usage is infrequent (<4 hours/day)
4. **Hybrid Approach**: Use CloudFront + API Gateway + ALB for best of both

### Recommendation

- **Use Standard (ALB Always-On)** if:
  - You access the system daily
  - You need fast response times (<3 seconds)
  - You have predictable usage patterns
  - Cost difference ($23/month) is acceptable

- **Use True Zero-Cost** if:
  - You access the system weekly or less
  - You can tolerate 5-10 minute cold starts
  - You want absolute minimum costs
  - You're okay with "warming up" messages

## Migration Path to CDK

The Terraform structure is designed to facilitate migration to AWS CDK:

1. **Module Mapping**: Each Terraform module maps to a CDK construct
2. **State Management**: Use CDK's built-in state management
3. **Gradual Migration**: Migrate module by module
4. **Import Existing Resources**: Use CDK import to adopt Terraform-created resources

## Security Considerations

1. **Network Isolation**: Private subnets for all application components
2. **Secrets Management**: No hardcoded secrets, use AWS Secrets Manager
3. **IAM Roles**: Least privilege principle for ECS tasks
4. **Encryption**: 
   - EFS encryption at rest
   - S3 encryption (SSE-S3)
   - Secrets Manager encryption
5. **Security Groups**: Restrictive ingress/egress rules
6. **VPC Flow Logs**: Enabled for network monitoring

## Backup & Disaster Recovery

### Automated Backups
- **MySQL**: Continuous sync to S3 (every 15 minutes) + daily full backups
- **Redis**: Hourly RDB snapshots to S3
- **Application Files**: S3 versioning enabled
- **Retention**: 30 days for all S3 backups

### Recovery Procedures
1. **Database Recovery**: Restore from S3 backup (15-minute incremental or daily full)
2. **Application Recovery**: Redeploy from ECR images
3. **Configuration Recovery**: Terraform state in S3
4. **RTO**: ~10 minutes
5. **RPO**: ~15 minutes (continuous S3 sync)

## Phase 2: Development Container

### Devcontainer Configuration

```json
{
  "name": "OK.py AWS Development",
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "features": {
    "ghcr.io/devcontainers/features/aws-cli:1": {},
    "ghcr.io/devcontainers/features/terraform:1": {},
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "hashicorp.terraform",
        "amazonwebservices.aws-toolkit-vscode"
      ]
    }
  },
  "mounts": [
    "source=${localEnv:HOME}/.aws,target=/home/vscode/.aws,type=bind"
  ]
}
```

## Success Criteria

- [ ] Infrastructure deploys successfully via Terraform
- [ ] Application accessible via ALB URL
- [ ] Scale-to-zero works automatically after 30 minutes of inactivity
- [ ] Manual scale-up completes in under 3 minutes
- [ ] Database data persists across tear-downs
- [ ] All operations executable via Docker containers
- [ ] Monthly cost under $60 with optimization
- [ ] Backup and restore procedures validated
- [ ] Documentation complete and tested

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| S3 sync latency for MySQL | Medium | Use 15-minute sync interval, implement write-ahead logging |
| MySQL data loss between syncs | Medium | Implement transaction logs, reduce sync interval if needed |
| NAT Gateway single point of failure | Low | Accept for cost optimization, add second NAT if needed |
| Cold start time too long | Medium | Keep ALB warm with health checks, optimize container startup |
| MySQL container stability | High | Implement robust health checks, auto-restart, S3 backup before restart |
| Cost overruns | High | Set up billing alarms, regular cost reviews, aggressive optimization |

## Next Steps

1. Create work decomposition and task tracking
2. Set up project structure
3. Implement Terraform modules
4. Create deployment scripts
5. Build Docker tooling containers
6. Test deployment in dev environment
7. Document operations procedures
8. Create devcontainer configuration
