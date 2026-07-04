# AWS Deployment Work Breakdown Structure

## Project Overview
Deploy ok.py to AWS with cost-optimized, scale-to-zero infrastructure using Terraform and containerized tooling.

## Epic 1: Project Setup & Foundation
**Estimated Effort**: 2-3 days

### Task 1.1: Repository Structure Setup
- [ ] Create `infrastructure/` directory structure
- [ ] Create `infrastructure/docker/` for CLI containers
- [ ] Create `infrastructure/terraform/` with module structure
- [ ] Create `infrastructure/scripts/` for deployment scripts
- [ ] Create `infrastructure/lambda/` for scale functions
- [ ] Add `.gitignore` for Terraform state files
- [ ] Create `infrastructure/docs/` directory
**Acceptance Criteria**: Directory structure matches tech spec

### Task 1.2: Docker CLI Tooling Containers
- [ ] Create `Dockerfile.terraform` with Terraform 1.6+
- [ ] Create `Dockerfile.aws-cli` with AWS CLI v2
- [ ] Create `Dockerfile.tools` combining both
- [ ] Add docker-compose.yml for local development
- [ ] Create `scripts/build-tools.sh` script
- [ ] Test container builds locally
- [ ] Document container usage in README
**Acceptance Criteria**: All containers build successfully, AWS SSO works in container

### Task 1.3: Terraform Backend Setup
- [ ] Create S3 bucket for Terraform state (manual or script)
- [ ] Create DynamoDB table for state locking
- [ ] Create `backend.tf` configuration
- [ ] Create `versions.tf` with provider versions
- [ ] Create `scripts/init-terraform.sh` script
- [ ] Test backend initialization
- [ ] Document backend setup process
**Acceptance Criteria**: Terraform state stored in S3 with locking

## Epic 2: Core Infrastructure Modules
**Estimated Effort**: 5-7 days

### Task 2.1: Networking Module
- [ ] Create `modules/networking/main.tf`
- [ ] Define VPC with 10.0.0.0/16 CIDR
- [ ] Create 2 public subnets (AZ-a, AZ-b)
- [ ] Create 2 private subnets (AZ-a, AZ-b)
- [ ] Create Internet Gateway
- [ ] Create NAT Gateway (single, cost-optimized)
- [ ] Create route tables and associations
- [ ] Define security groups (ALB, ECS, MySQL, Redis)
- [ ] Create `modules/networking/variables.tf`
- [ ] Create `modules/networking/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: VPC created with proper subnet routing, security groups configured

### Task 2.2: ECS Cluster Module
- [ ] Create `modules/ecs/main.tf`
- [ ] Define ECS cluster resource
- [ ] Create CloudWatch log groups
- [ ] Define ECS task execution role
- [ ] Define ECS task role with S3/Secrets Manager permissions
- [ ] Create `modules/ecs/variables.tf`
- [ ] Create `modules/ecs/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: ECS cluster created with proper IAM roles

### Task 2.3: Storage Module (S3-Only, No EFS)
- [ ] Create `modules/storage/main.tf`
- [ ] Define S3 bucket for application storage
- [ ] Define S3 bucket for backups
- [ ] Define S3 bucket for MySQL data (`ok-mysql-data-{env}`)
- [ ] Define S3 bucket for Redis snapshots (`ok-redis-data-{env}`)
- [ ] Configure S3 lifecycle policies (Intelligent-Tiering)
- [ ] Enable S3 versioning on all buckets
- [ ] Configure S3 bucket policies for ECS task access
- [ ] Create `modules/storage/variables.tf`
- [ ] Create `modules/storage/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: All S3 buckets created with proper policies, no EFS resources

### Task 2.4: Database Module (MySQL Container with S3 Sync)
- [ ] Create `modules/database/main.tf`
- [ ] Define ECS task definition for MySQL 8.0
- [ ] Configure S3 sync sidecar container (s3fs-fuse or custom script)
- [ ] Set up continuous backup to S3 (every 15 minutes)
- [ ] Define MySQL environment variables
- [ ] Create ECS service for MySQL
- [ ] Configure service discovery (optional)
- [ ] Define security group rules
- [ ] Create health check configuration
- [ ] Add S3 restore on container startup
- [ ] Create `modules/database/variables.tf`
- [ ] Create `modules/database/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: MySQL container runs with continuous S3 sync, data persists across restarts

### Task 2.5: Cache Module (Redis Container with S3 Snapshots)
- [ ] Create `modules/cache/main.tf`
- [ ] Define ECS task definition for Redis 7.0
- [ ] Configure RDB snapshot to S3 (hourly via cron)
- [ ] Define Redis environment variables
- [ ] Create ECS service for Redis
- [ ] Configure service discovery (optional)
- [ ] Define security group rules
- [ ] Create health check configuration
- [ ] Add S3 restore on container startup
- [ ] Create `modules/cache/variables.tf`
- [ ] Create `modules/cache/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: Redis container runs with hourly S3 snapshots, data persists across restarts

### Task 2.6: Application Load Balancer Module
- [ ] Create `modules/alb/main.tf`
- [ ] Define Application Load Balancer
- [ ] Create target group for ok-web service
- [ ] Configure health check on `/healthz`
- [ ] Create HTTP listener (port 80)
- [ ] Create HTTPS listener (port 443) - optional
- [ ] Configure listener rules
- [ ] Create `modules/alb/variables.tf`
- [ ] Create `modules/alb/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: ALB created with proper target groups and health checks

## Epic 3: Application Services
**Estimated Effort**: 4-5 days

### Task 3.1: ECR Repositories
- [ ] Create `modules/ecr/main.tf`
- [ ] Define ECR repository for ok-web
- [ ] Define ECR repository for ok-worker
- [ ] Configure lifecycle policies
- [ ] Create `modules/ecr/variables.tf`
- [ ] Create `modules/ecr/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: ECR repositories created with lifecycle policies

### Task 3.2: Docker Image Build Pipeline
- [ ] Create `scripts/build-images.sh`
- [ ] Configure Docker build for ok-web
- [ ] Configure Docker build for ok-worker
- [ ] Add AWS ECR authentication
- [ ] Add image tagging strategy (git commit, latest)
- [ ] Add image push to ECR
- [ ] Test build process locally
- [ ] Document build process
**Acceptance Criteria**: Images build and push to ECR successfully

### Task 3.3: OK-Web ECS Service
- [ ] Create `modules/app-web/main.tf`
- [ ] Define ECS task definition for ok-web
- [ ] Configure environment variables from Secrets Manager
- [ ] Configure S3 storage backend
- [ ] Set resource limits (0.25 vCPU, 512 MB)
- [ ] Create ECS service with ALB integration
- [ ] Configure service discovery
- [ ] Define security group rules
- [ ] Create `modules/app-web/variables.tf`
- [ ] Create `modules/app-web/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: ok-web service runs and responds to ALB health checks

### Task 3.4: OK-Worker ECS Service
- [ ] Create `modules/app-worker/main.tf`
- [ ] Define ECS task definition for ok-worker
- [ ] Configure environment variables from Secrets Manager
- [ ] Configure Redis connection
- [ ] Set resource limits (0.25 vCPU, 512 MB)
- [ ] Create ECS service
- [ ] Define security group rules
- [ ] Create `modules/app-worker/variables.tf`
- [ ] Create `modules/app-worker/outputs.tf`
- [ ] Add module documentation
**Acceptance Criteria**: ok-worker service runs and processes jobs from Redis

### Task 3.5: Secrets Management
- [ ] Create `modules/secrets/main.tf`
- [ ] Define Secrets Manager secrets for:
  - Database credentials
  - OAuth client secrets (Google, Microsoft)
  - SendGrid API key
  - Session secret key
  - Storage credentials
- [ ] Create IAM policy for ECS tasks to read secrets
- [ ] Create `modules/secrets/variables.tf`
- [ ] Create `modules/secrets/outputs.tf`
- [ ] Document secret rotation process
**Acceptance Criteria**: All secrets stored in Secrets Manager, accessible by ECS tasks

## Epic 4: Auto-Scaling & Cost Optimization
**Estimated Effort**: 3-4 days

### Task 4.0: Architecture Decision - Standard vs True Zero-Cost
- [ ] Review usage patterns and requirements
- [ ] Evaluate cost vs performance trade-offs
- [ ] Decide on Standard (ALB always-on) or True Zero-Cost (API Gateway)
- [ ] Document decision and rationale
- [ ] Update deployment plan accordingly
**Acceptance Criteria**: Architecture decision documented with clear rationale

### Task 4.1: Application Auto-Scaling
- [ ] Create `modules/autoscaling/main.tf`
- [ ] Define auto-scaling target for ok-web
- [ ] Configure target tracking policy (CPU 70%)
- [ ] Configure target tracking policy (Memory 80%)
- [ ] Configure step scaling based on ALB requests
- [ ] Set scale-in/scale-out cooldowns
- [ ] Create `modules/autoscaling/variables.tf`
- [ ] Create `modules/autoscaling/outputs.tf`
- [ ] Test auto-scaling behavior
**Acceptance Criteria**: Services scale up/down based on load

### Task 4.2: Scale-to-Zero Lambda Function
- [ ] Create `lambda/scale-to-zero/index.py`
- [ ] Implement ECS service scaling to 0
- [ ] Add CloudWatch Logs integration
- [ ] Create `lambda/scale-to-zero/requirements.txt`
- [ ] Create Lambda deployment package
- [ ] Create `modules/lambda/scale-to-zero.tf`
- [ ] Define Lambda IAM role with ECS permissions
- [ ] Create CloudWatch alarm for no requests (30 min)
- [ ] Connect alarm to Lambda trigger
- [ ] Test scale-to-zero functionality
**Acceptance Criteria**: Services scale to 0 after 30 minutes of inactivity

### Task 4.3: Scale-Up Lambda Function
- [ ] Create `lambda/scale-up/index.py`
- [ ] Implement ECS service scaling to 1
- [ ] Add CloudWatch Logs integration
- [ ] Create `lambda/scale-up/requirements.txt`
- [ ] Create Lambda deployment package
- [ ] Create `modules/lambda/scale-up.tf`
- [ ] Define Lambda IAM role with ECS permissions
- [ ] Create ALB target registration trigger
- [ ] Test scale-up functionality
**Acceptance Criteria**: Services scale up when ALB receives requests

### Task 4.4: Cost Monitoring
- [ ] Create `modules/monitoring/billing.tf`
- [ ] Define CloudWatch billing alarms
- [ ] Set budget alerts ($60/month threshold)
- [ ] Create SNS topic for cost alerts
- [ ] Configure email notifications
- [ ] Document cost optimization strategies
**Acceptance Criteria**: Billing alarms trigger when costs exceed threshold

## Epic 5: Monitoring & Logging
**Estimated Effort**: 2-3 days

### Task 5.1: CloudWatch Logs
- [ ] Create `modules/monitoring/logs.tf`
- [ ] Define log groups for all services
- [ ] Set log retention to 7 days
- [ ] Configure log streaming from ECS tasks
- [ ] Create log insights queries
- [ ] Document log access procedures
**Acceptance Criteria**: All service logs available in CloudWatch

### Task 5.2: CloudWatch Metrics & Alarms
- [ ] Create `modules/monitoring/alarms.tf`
- [ ] Define alarms for:
  - High CPU usage (>80%)
  - High memory usage (>80%)
  - Health check failures
  - No requests (scale-to-zero trigger)
- [ ] Create SNS topic for alarm notifications
- [ ] Configure email notifications
- [ ] Test alarm triggers
**Acceptance Criteria**: Alarms trigger and send notifications

### Task 5.3: Monitoring Dashboard
- [ ] Create `modules/monitoring/dashboard.tf`
- [ ] Define CloudWatch dashboard with:
  - ECS task count
  - ALB request count
  - Database connections
  - Redis memory usage
  - Cost metrics
- [ ] Add custom metrics
- [ ] Document dashboard usage
**Acceptance Criteria**: Dashboard displays all key metrics

## Epic 6: Backup & Disaster Recovery
**Estimated Effort**: 2-3 days

### Task 6.1: Automated Backup Configuration
- [ ] Create `modules/backup/main.tf`
- [ ] Configure S3 lifecycle policies for backups
- [ ] Set backup retention to 30 days
- [ ] Configure S3 versioning for point-in-time recovery
- [ ] Define backup IAM roles for ECS tasks
- [ ] Create `modules/backup/variables.tf`
- [ ] Create `modules/backup/outputs.tf`
- [ ] Test backup creation and restoration
**Acceptance Criteria**: Continuous S3 backups with 30-day retention

### Task 6.2: Backup Scripts
- [ ] Create `scripts/backup.sh` for manual backups
- [ ] Implement MySQL dump to S3
- [ ] Implement Redis RDB snapshot to S3
- [ ] Add backup verification
- [ ] Document backup procedures
**Acceptance Criteria**: Manual backup script works correctly

### Task 6.3: Restore Scripts
- [ ] Create `scripts/restore.sh` for recovery
- [ ] Implement MySQL restore from S3
- [ ] Implement Redis restore from S3 snapshot
- [ ] Add restore verification
- [ ] Document restore procedures
- [ ] Test full restore process
**Acceptance Criteria**: Restore script successfully recovers data from S3

## Epic 7: Deployment Automation
**Estimated Effort**: 3-4 days

### Task 7.1: Environment Configuration
- [ ] Create `terraform/environments/dev/main.tf`
- [ ] Create `terraform/environments/dev/variables.tf`
- [ ] Create `terraform/environments/dev/terraform.tfvars`
- [ ] Create `terraform/environments/dev/outputs.tf`
- [ ] Repeat for staging environment
- [ ] Repeat for prod environment
- [ ] Document environment differences
**Acceptance Criteria**: All environments configured with appropriate settings

### Task 7.2: Deployment Scripts
- [ ] Create `scripts/deploy.sh` main deployment script
- [ ] Add environment validation
- [ ] Add Terraform plan review
- [ ] Add Terraform apply with approval
- [ ] Add post-deployment verification
- [ ] Add rollback capability
- [ ] Test deployment to dev environment
- [ ] Document deployment process
**Acceptance Criteria**: Deployment script successfully deploys infrastructure

### Task 7.3: Tear-Down Scripts
- [ ] Create `scripts/destroy.sh` tear-down script
- [ ] Add data preservation checks
- [ ] Add confirmation prompts
- [ ] Add backup creation before destroy
- [ ] Add Terraform destroy execution
- [ ] Test tear-down in dev environment
- [ ] Document tear-down process
**Acceptance Criteria**: Tear-down script preserves data and destroys infrastructure

### Task 7.4: Operational Scripts
- [ ] Create `scripts/scale-down.sh` for manual scale-down
- [ ] Create `scripts/scale-up.sh` for manual scale-up
- [ ] Create `scripts/logs.sh` for log viewing
- [ ] Create `scripts/status.sh` for infrastructure status
- [ ] Create `scripts/ssh.sh` for ECS Exec access
- [ ] Test all operational scripts
- [ ] Document script usage
**Acceptance Criteria**: All operational scripts work correctly

## Epic 8: Documentation & Testing
**Estimated Effort**: 2-3 days

### Task 8.1: Deployment Documentation
- [ ] Create `infrastructure/docs/DEPLOYMENT.md`
- [ ] Document prerequisites
- [ ] Document initial setup steps
- [ ] Document deployment workflow
- [ ] Add troubleshooting section
- [ ] Add FAQ section
- [ ] Review and test documentation
**Acceptance Criteria**: Documentation is complete and accurate

### Task 8.2: Operations Runbook
- [ ] Create `infrastructure/docs/OPERATIONS.md`
- [ ] Document daily operations
- [ ] Document monitoring procedures
- [ ] Document backup/restore procedures
- [ ] Document scaling procedures
- [ ] Document incident response
- [ ] Add common issues and solutions
**Acceptance Criteria**: Runbook covers all operational scenarios

### Task 8.3: Cost Optimization Guide
- [ ] Create `infrastructure/docs/COST_OPTIMIZATION.md`
- [ ] Document cost breakdown
- [ ] Document optimization strategies
- [ ] Document scale-to-zero configuration
- [ ] Add cost monitoring setup
- [ ] Add cost reduction tips
**Acceptance Criteria**: Guide provides actionable cost optimization strategies

### Task 8.4: Integration Testing
- [ ] Test full deployment from scratch
- [ ] Test application functionality
- [ ] Test auto-scaling behavior
- [ ] Test scale-to-zero functionality
- [ ] Test backup and restore
- [ ] Test tear-down and redeploy
- [ ] Document test results
**Acceptance Criteria**: All tests pass successfully

## Epic 9: Phase 2 - Development Container
**Estimated Effort**: 1-2 days

### Task 9.1: Devcontainer Configuration
- [ ] Create `.devcontainer/devcontainer.json`
- [ ] Create `.devcontainer/Dockerfile`
- [ ] Add AWS CLI feature
- [ ] Add Terraform feature
- [ ] Add Docker-in-Docker feature
- [ ] Configure VS Code extensions
- [ ] Configure AWS SSO mount
- [ ] Test devcontainer locally
- [ ] Document devcontainer usage
**Acceptance Criteria**: Devcontainer provides full development environment

### Task 9.2: Development Workflow Documentation
- [ ] Create `infrastructure/docs/DEVELOPMENT.md`
- [ ] Document devcontainer setup
- [ ] Document local testing procedures
- [ ] Document debugging procedures
- [ ] Add development best practices
**Acceptance Criteria**: Developers can set up and use devcontainer

## Project Milestones

### Milestone 1: Foundation Complete (Week 1)
- Project structure created
- Docker tooling containers built
- Terraform backend initialized

### Milestone 2: Core Infrastructure (Week 2-3)
- Networking module deployed
- ECS cluster created
- Storage configured
- Database and cache running

### Milestone 3: Application Deployed (Week 3-4)
- ECR repositories created
- Docker images built and pushed
- ok-web and ok-worker services running
- Application accessible via ALB

### Milestone 4: Production Ready (Week 4-5)
- Auto-scaling configured
- Scale-to-zero working
- Monitoring and logging complete
- Backup and restore tested
- Documentation complete

### Milestone 5: Phase 2 Complete (Week 5-6)
- Devcontainer configured
- Development workflow documented
- Team onboarded

## Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| MySQL container instability | Medium | High | Implement robust health checks, auto-restart policies |
| EFS performance issues | Low | Medium | Monitor performance, upgrade to provisioned throughput if needed |
| Scale-to-zero cold start too slow | Medium | Medium | Optimize container startup, consider keeping ALB warm |
| Cost overruns | Medium | High | Set up billing alarms, regular cost reviews, aggressive optimization |
| Terraform state corruption | Low | High | Use S3 versioning, regular state backups |
| AWS SSO authentication issues | Low | Medium | Document SSO setup, provide fallback authentication |

## Success Metrics

- [ ] Infrastructure deploys in under 15 minutes
- [ ] Application responds to requests in under 3 seconds
- [ ] Scale-to-zero activates after 30 minutes of inactivity
- [ ] Scale-up completes in under 3 minutes
- [ ] Monthly cost under $60 with optimization
- [ ] Zero data loss during tear-down/restore
- [ ] All operations executable via Docker containers
- [ ] 100% documentation coverage

## Total Estimated Effort
- **Total Days**: 24-34 days
- **Total Weeks**: 5-7 weeks (assuming 5 days/week)
- **Recommended Timeline**: 6 weeks with buffer for testing and refinement
