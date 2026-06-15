# Round-3--Assessment
───────────────────────────────────────────────────────────────────────────────────────────
TASK 1 — AWS Infrastructure & Cost Optimization
───────────────────────────────────────────────────────────────────────────────────────────
Your team has a staging environment on AWS running 24/7 with ECS Fargate services, an RDS instance, and an ALB. The monthly bill has crossed $600 and your manager wants it cut by at least 40% without breaking the environment.

Deliverable:
* A shell script or Terraform snippet that implements an auto start/stop schedule for the RDS instance (stop nights + weekends)
* A short written list (5–7 points) of additional cost optimisation steps across ECS, ALB, and data transfer

───────────────────────────────────────────────────────────────────────────────────────────
Solution:
Auto Start/Stop Schedule for the RDS:
*Shell script (AWS CLI) is executed from a scheduled EC2 instance or CI/CD pipeline.

Stop RDS (night + weekends):

#!/bin/bash
DB_INSTANCE_ID="staging-db"
echo "Stopping RDS instance: $DB_INSTANCE_ID"
aws rds stop-db-instance --db-instance-identifier $DB_INSTANCE_ID
echo "Stop request submitted."

Start RDS (weekday mornings):

#!/bin/bash
DB_INSTANCE_ID="staging-db"
echo "Starting RDS instance: $DB_INSTANCE_ID"
aws rds start-db-instance --db-instance-identifier $DB_INSTANCE_ID
echo "Start request submitted."

Example Cron Schedule:

Start every weekday @ 8AM:
0 8 * * 1 - 5 /stop-rds.sh
Stop every weekday @8PM:
0 20 * * 1 -5 /stop-rds.sh
Stop Friday night and keep stopped through weekend:
0 20 * * 5 /stop-rds.sh
───────────────────────────────────────────────────────────────────────────────────────────
Additional Cost Optimization Steps:

1.	Scale ECS Fargate Tasks by Schedule and Run fewer tasks during non-working hours:
Saving 30% more ECS compute costs.
2.	Use Fargate Spot for non-critical staging workloads:
Upto 70% less compute cost compared with std Fargate.
3.	Right-size CPU and Memory allocations and review ECS task definitions:
CloudWatch metrics to reduce CPU and Memory, saving 20% more cost.
4.	Shut Down ECS services outside working hours if staging is not needed overnight. Restart in the morning:
Saving nearing 100% of ECS runtime costs during off-hours.
5.	Remove unused ALBs and check whether multiple staging apps can share one ALB using host-based/path-based routing:
Eliminates duplicate ALB charges and reduces LCU costs.
6.	Reduce Data transfer charges:
Keep ECS and RDS in the same AZ when acceptable for staging,
Use VPC endpoints, 
Minimize NAT Gateway usage.
7.	Enable Log retention policies as CloudWatch logs often grow unnoticed:
Reduces log storage costs
────────────────────────────────────────────────────────────────────────────────────────
Expected Cost Reduction Summary:
Optimisation              Saving
RDS schedule stop/start    15%
ECS task scheduling        10%
Fargate Spot               10%
Right-sizing ECS resources 10%
ALB consolidation	         2%
Data transfer optimisation 2%
Log retention cleanup      1%

Combined saving can realistically exceed the required 40% cut in cost without breaking the current staging environment during working hours.


───────────────────────────────────────────────────────────────────────────────────────────
TASK 2 — CI/CD Pipeline Design
───────────────────────────────────────────────────────────────────────────────────────────
Scenario:
A Node.js backend application is currently deployed manually via SSH. A GitHub repo, ECR registry, and ECS Fargate cluster are already in place. You've been asked to set up a proper CI/CD pipeline from scratch.

Deliverable:
A working GitHub Actions YAML file that:
* Triggers on push to main
* Builds and pushes a Docker image to ECR with a git SHA tag
* Updates the ECS service to force a new deployment
* Includes inline comments explaining each step

───────────────────────────────────────────────────────────────────────────────────────────
Solution:
File: .github/workflows/deploy.yml

name: Build and Deploy to ECS 

#Trigger workflow whenever code is pushed to the main branch
on:
  push:
    branches:
      - main

jobs:
  deploy:
    name: Build, Push to ECR, and Deploy to ECS
    runs-on: ubuntu-latest

    env:
      AWS_REGION: us-east-1
      ECR_REPOSITORY: my-nodejs-backend
      ECS_CLUSTER: my-ecs-cluster
      ECS_SERVICE: my-ecs-service
    
    steps:
      
      # Step 1: Download 	repository source code
      - name: Checkout source code
        uses: actions/checkout@v4
        
      # Step 2: Configure AWS Credentials
      # Store these values in GitHub Repository Secrets:
      # AWS_ACCESS_KEY_ID
      # AWS_SECRET_ACCESS_KEY
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      # Step 3: Authenticate Docker to Amazon ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Step 4: Create a Docker Image tag using the Git commit SHA
      - name: Generate Image Tag
        run: echo "IMAGE_TAG=${GITHUB_SHA}" >> $GITHUB_ENV

      # Step 5: Build the Docker Image
      - name: Build Docker Image
        run: docker build -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest .

      # Step 6: Push the Docker Image to ECR
      - name: Push Docker Image to ECR
        run: docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest

      # Step 7: Force ECS Service to Start a New Deployment
      # ECS will pull the latest image and replace running tasks
      - name: Deploy to ECS
        run: aws ecs update-service --cluster ${{ env.ECS_CLUSTER }} --service ${{ env.ECS_SERVICE }} --force-new-deployment

      # Step 8: Wait until deployment finishes successfully
      - name: Wait for ECS Deployment
        run: aws ecs wait services-stable --cluster ${{ env.ECS_CLUSTER }} --services ${{ env.ECS_SERVICE }}

      # Step 9: Output deployment information
      - name: Deployment Complete
        run: |
          echo "Deployment successful"
          echo "Image Tag: ${IMAGE_TAG}"


───────────────────────────────────────────────────────────────────────────────────────────
TASK 3 — Incident Troubleshooting (Containers + Networking)
───────────────────────────────────────────────────────────────────────────────────────────
Scenario:
A containerised application on ECS Fargate is returning 503 Service Unavailable intermittently via the ALB. Tasks show as RUNNING, CloudWatch shows no CPU/memory spikes, and app logs look clean. The issue started after a recent deployment.

Deliverable:
A step-by-step troubleshooting runbook covering:
* Where you would look first and why
* Specific AWS CLI commands or console checks at each step
* At least 3 plausible root causes with confirmation/ruling-out steps
* How you'd roll back if needed

───────────────────────────────────────────────────────────────────────────────────────────
Solution:
1. Initial Triage - Where to look first:
   Since the issue appeared immidiately after the deployment and tasks remain healthy from an ECS perspective, the first sus is:
   1. ALB Target Group health issues
   2. Port/container mapping changes
   3. Health check failures
   4. Security group changes
   5. New task definition deployment issues

   The Objective is to determine whether:
   * ALB cannot reach tasks
   * Tasks are failing health checks
   * ECS deployed an incorrect configuration
   * Traffic routing is inconsistent

───────────────────────────────────────────────────────────────────────────────────────────
Step 1: Check ALB Target Health
Reason:  An ALB returns 503 when it has no healthy targets available for a request
AWS CLI: aws elbv2 describe-target-health --target-group-arn <TARGET_GROUP_ARN>
Console: AWS Console > EC2 > Target Groups > Targets
Check:   Healthy target counts
         Unhealthy target counts
         Health status reason codes

Step 2: Verify ECS Service Events
Reason:  ECS service events often immediately reveal deployment failures.
AWS CLI: aws ecs describe-services --cluster my-cluster --services my-service
Console: ECS > Cluster > Service > Events
Check:   * Failed targets registration
         * Health check failures
         * Task replacement loops
         * Insufficient resources

Step 3: Compare Current vs Previous Task definition
Reason:  Incident started after deployment. Most ECS outages originate from task definition changes.
AWS CLI: Current revision - aws ecs describe-services --cluster my-cluster --services my-service
         Inspect task definition: aws ecs describe-task-definition --task-definition my-task:42
         Compare against previous revision: aws ecs describe-task-definition --task-definition my-task:41
Check:   * Container port changes
         * Health endpoint changes
         * Environment variable changes
         * Secrets changes
         * Startup command changes

Step 4: Validate ALB Health Check Configuration
Reason:  Deployment may have changed the application path/port.
AWS CLI: aws elbv2 describe-target-groups --target-group-arns <TARGET_GROUP_ARN>
Check:   * Health check path
         * Health check port
         * Matcher (200, 301, etc)
         * Interval
         * Timeout

Step 5: Verify Port Mappings
Reason:  most common post-deployment issues is a mismatched ECS container port, Application listening port, or Target group port.
AWS CLI: aws ecs describe-task-definition --task-defnition my-task:42
Example: "portMappings": [
           {
             "containerPort": 3000
           }
         ]
Check:   application actually listens on:
         netstat -tulpn
         or
         ss -tulpn
Expected: 0.0.0.0:3000

Step 6: Check Security Groups:
Reason:  Tasks may be healthy internally but unreachable from ALB.
ALB Security Group: Must allow Inbound - 80/443 from internet
ECS Task Security group: Must allow Inbound - Application port source (ALB Security group)
AWS CLI: aws ec2 describe-security-groups --group-ids sg-xxxx
Check:   * ALB SG > ECS SG allowed
         * No recent changes

Step 7: Check ALB Access Logs
Reason: Confirms exactly why requests fail. Enables ALB access logs if not already enabled.
Check:  target_status_code
        elb_status_code
Examples: 503 - TargetConnectionError
       or 503 - NoHealthyTargets
These immediately narrow the investigation.

Step 8: Inspect ECS Task Networking
Reason:  Fargate uses aws vpc networking. Deployment may have altered: * Subnets
                                                                       * Route tables
                                                                       * Security groups
AWS CLI: aws ecs describe-tasks --cluster my-cluster --tasks <TASK_ID>
Check:   * Correct subnet
         * Correct security group
         * ENI attached successfully

Step 9: Validate Application Readiness
Reason:  Container may start successfully but not actually be ready
         ECS = Running, but ALB health check fail intermittently
AWS CLI: aws logs tail /ecs/my-app --follow
Check:   * Slow startup
         * Database connection delays
         * Dependancy initialization failures

───────────────────────────────────────────────────────────────────────────────────────────
2. Pausible Root Causes
Root Cause 1: Health Check Endpoint Changed
