# ECS Exercise - Complete Console-Based Tutorial

## Prerequisites
- AWS Account with appropriate permissions
- AWS CLI configured (optional)

## Step 1: Deploy VPC Infrastructure

1. **Deploy CloudFormation Template**
   - Go to AWS Console → CloudFormation
   - Click "Create stack" → "With new resources"
   - Upload the `ecs-exercise-vpc.yml` template
   - Stack name: `ecs-exercise-vpc`
   - Click "Next" → "Next" → "Create stack"
   - Wait for stack to complete (~5-10 minutes)

## Step 2: Create ECS Cluster (Latest Features)

1. **Navigate to ECS Console**
   - Go to AWS Console → Elastic Container Service

2. **Create Cluster**
   - Click "Create Cluster"
   - **Cluster name**: `my-ecs-cluster`
   - **Infrastructure**: Select "AWS Fargate (serverless)"
   - **Monitoring**: Enable Container Insights (optional)
   - **Tags**: Add any desired tags
   - Click "Create"

## Step 3: Create Application Load Balancer

1. **Navigate to EC2 Console → Load Balancers**
   - Click "Create Load Balancer"
   - Select "Application Load Balancer"

2. **Configure Load Balancer**
   - **Name**: `ecs-exercise-alb`
   - **Scheme**: Internet-facing
   - **IP address type**: IPv4
   - **VPC**: Select the VPC created by CloudFormation
   - **Mappings**: Select both public subnets
   - **Security groups**: Select the ALB security group from CloudFormation
   - **Listeners**: HTTP:80 (create dummy target group for now)
   - Click "Create load balancer"

## Step 4: Create ECR Repositories

1. **Navigate to ECR Console**
   - Go to AWS Console → Elastic Container Registry

2. **Create Repository for Web App**
   - Click "Create repository"
   - **Repository name**: `web-app`
   - **Tag immutability**: Disabled
   - **Scan on push**: Enabled
   - Click "Create repository"

3. **Create Repository for API**
   - Repeat above steps with name: `api-app`

## Step 5: Build and Push Sample Images

1. **Get Login Command**
   - In ECR console, select `web-app` repository
   - Click "View push commands"
   - Follow the commands to authenticate Docker
   - **Note**: Add `sudo` before docker login if you get permission errors:
   ```bash
   aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin [ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com
   ```

2. **Create Simple Web App**
   ```bash
   # Create Dockerfile for web app
   mkdir web-app && cd web-app
   cat > Dockerfile << EOF
   FROM nginx:alpine
   COPY index.html /usr/share/nginx/html/
   EXPOSE 80
   EOF

   cat > index.html << EOF
   <!DOCTYPE html>
   <html>
   <head><title>ECS Web App</title></head>
   <body>
       <h1>Hello from ECS Web App!</h1>
       <p>Running on Fargate</p>
   </body>
   </html>
   EOF
   ```

5. **Build and Push Web App**
   ```bash
   # Note: Add 'sudo' before docker commands if you get permission errors
   sudo docker build -t web-app .
   sudo docker tag web-app:latest [ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com/web-app:latest
   sudo docker push [ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com/web-app:latest
   ```

4. **Create Simple API App**
   ```bash
   cd .. && mkdir api-app && cd api-app
   cat > Dockerfile << EOF
   FROM node:alpine
   WORKDIR /app
   COPY package.json app.js ./
   RUN npm install
   EXPOSE 3000
   CMD ["node", "app.js"]
   EOF

   cat > package.json << EOF
   {
     "name": "api-app",
     "version": "1.0.0",
     "dependencies": {
       "express": "^4.18.0"
     }
   }
   EOF

   cat > app.js << EOF
   const express = require('express');
   const app = express();
   
   app.get('/health', (req, res) => {
     res.json({ status: 'healthy', service: 'api-app' });
   });
   
   app.get('/api/data', (req, res) => {
     res.json({ message: 'Hello from ECS API!', timestamp: new Date() });
   });
   
   app.listen(3000, () => {
     console.log('API server running on port 3000');
   });
   EOF
   ```

5. **Build and Push API App**
   ```bash
   # Note: Add 'sudo' before docker commands if you get permission errors
   sudo docker build -t api-app .
   sudo docker tag api-app:latest [ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com/api-app:latest
   sudo docker push [ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com/api-app:latest
   ```

## Step 6: Create Task Definitions

1. **Create Web App Task Definition**
   - Go to ECS Console → Task Definitions
   - Click "Create new task definition"
   - **Family name**: `web-app-task`
   - **Launch type**: Fargate
   - **Operating system**: Linux/X86_64
   - **CPU**: 0.25 vCPU
   - **Memory**: 0.5 GB
   - **Task role**: None (or create if needed)
   - **Task execution role**: ecsTaskExecutionRole

2. **Add Container to Web App Task**
   - **Container name**: `web-container`
   - **Image URI**: `[ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com/web-app:latest`
   - **Port mappings**: Container port 80, Protocol TCP
   - **Environment variables**: None
   - **Log configuration**: 
     - Log driver: awslogs
     - Log group: `/ecs/web-app-task` (auto-create)
   - Click "Create"

3. **Create API App Task Definition**
   - Repeat above steps with:
   - **Family name**: `api-app-task`
   - **Container name**: `api-container`
   - **Image URI**: `[ACCOUNT-ID].dkr.ecr.[REGION].amazonaws.com/api-app:latest`
   - **Port mappings**: Container port 3000, Protocol TCP
   - **Log group**: `/ecs/api-app-task`

## Step 7: Create Target Groups

1. **Create Web App Target Group**
   - Go to EC2 Console → Target Groups
   - Click "Create target group"
   - **Target type**: IP addresses
   - **Target group name**: `web-app-tg`
   - **Protocol**: HTTP, Port: 80
   - **VPC**: Select your VPC
   - **Health check path**: `/`
   - Click "Create target group"

2. **Create API Target Group**
   - **Target group name**: `api-app-tg`
   - **Protocol**: HTTP, Port: 3000
   - **Health check path**: `/health`
   - Click "Create target group"

## Step 8: Update Load Balancer Listeners

1. **Configure ALB Listeners**
   - Go to EC2 Console → Load Balancers
   - Select your ALB → Listeners tab
   - Edit the HTTP:80 listener
   - **Default action**: Forward to `web-app-tg`

2. **Add API Listener Rule**
   - Click "Add rule"
   - **Conditions**: Path is `/api/*`
   - **Actions**: Forward to `api-app-tg`
   - **Priority**: 100

## Step 9: Create ECS Services

1. **Create Web App Service**
   - Go to ECS Console → Clusters → Select your cluster
   - Click "Create Service"
   - **Launch type**: Fargate
   - **Task Definition**: `web-app-task:1`
   - **Service name**: `web-app-service`
   - **Number of tasks**: 2
   - **Minimum healthy percent**: 50
   - **Maximum percent**: 200

2. **Configure Web App Networking**
   - **VPC**: Select your VPC
   - **Subnets**: Select private subnets
   - **Security groups**: Select ECS security group
   - **Auto-assign public IP**: Disabled
   - **Load balancer type**: Application Load Balancer
   - **Load balancer**: Select your ALB
   - **Target group**: `web-app-tg`
   - Click "Create Service"

3. **Create API Service**
   - Repeat above steps with:
   - **Task Definition**: `api-app-task:1`
   - **Service name**: `api-app-service`
   - **Target group**: `api-app-tg`

## Step 10: Test and Monitor

1. **Test Applications**
   - Get ALB DNS name from EC2 Console
   - Test web app: `http://[ALB-DNS]/`
   - Test API: `http://[ALB-DNS]/api/data`

2. **Monitor Services**
   - ECS Console → Clusters → Services
   - Check service status, task health
   - View CloudWatch logs
   - Monitor ALB target group health

## Step 11: ECS Auto Scaling Configuration

### A. Service Auto Scaling Setup

1. **Configure Target Tracking Scaling**
   - Go to ECS Console → Clusters → Select your cluster
   - Click on `web-app-service` → **Service auto scaling** tab
   - Click "Create" or "Configure auto scaling"
   - **Policy name**: `web-app-cpu-scaling`
   - **ECS service metric**: Average CPU utilization
   - **Target value**: 70%
   - **Scale-out cooldown**: 300 seconds
   - **Scale-in cooldown**: 300 seconds
   - **Minimum capacity**: 2
   - **Maximum capacity**: 10
   - Click "Create"

2. **Add Memory-Based Scaling**
   - Create another policy: `web-app-memory-scaling`
   - **ECS service metric**: Average Memory utilization
   - **Target value**: 80%
   - Same cooldown and capacity settings

3. **Configure API Service Scaling**
   - Repeat for `api-app-service` in **Service auto scaling** tab
   - **CPU target**: 60% (API typically more CPU intensive)
   - **Memory target**: 75%
   - **Min capacity**: 1, **Max capacity**: 5

### B. Step Scaling Configuration

1. **Create Step Scaling Policy**
   - In Auto Scaling tab, click "Create Auto Scaling policy"
   - **Policy type**: Step scaling
   - **Policy name**: `web-app-step-scaling`
   - **CloudWatch alarm**: Create new alarm
   - **Metric**: CPU Utilization
   - **Threshold**: Greater than 85%
   - **Period**: 2 minutes
   - **Evaluation periods**: 2

2. **Configure Scaling Steps**
   - **Step 1**: 85-95% CPU → Add 2 tasks
   - **Step 2**: 95%+ CPU → Add 4 tasks
   - **Scale-in steps**: 
     - 30-40% CPU → Remove 1 task
     - <30% CPU → Remove 2 tasks

### C. Scheduled Scaling

1. **Create Scheduled Actions**
   - Go to Application Auto Scaling console
   - Find your ECS service
   - Click "Create scheduled action"
   - **Name**: `morning-scale-up`
   - **Schedule**: `cron(0 8 * * MON-FRI)`
   - **Desired capacity**: 4
   - **Min capacity**: 2
   - **Max capacity**: 10

2. **Evening Scale Down**
   - **Name**: `evening-scale-down`
   - **Schedule**: `cron(0 18 * * MON-FRI)`
   - **Desired capacity**: 2

### D. Custom Metrics Scaling

1. **Create Custom CloudWatch Metric**
   - Go to CloudWatch → Metrics
   - Create custom metric for request count
   - Use ALB RequestCount metric as proxy

2. **Configure Custom Metric Scaling**
   - **Policy name**: `request-based-scaling`
   - **Metric**: ALB RequestCountPerTarget
   - **Target value**: 1000 requests per minute per task
   - **Scale-out**: When > 1000 RPM per task
   - **Scale-in**: When < 500 RPM per task

### E. Load Testing for Scaling

1. **Install Load Testing Tools on Ubuntu**
   ```bash
   # Update package list
   sudo apt update
   
   # Install Apache Bench
   sudo apt install apache2-utils -y
   
   # Install curl (usually pre-installed)
   sudo apt install curl -y
   
   # Install hey (alternative load testing tool)
   wget https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
   chmod +x hey_linux_amd64
   sudo mv hey_linux_amd64 /usr/local/bin/hey
   
   # Install wrk (another popular tool)
   sudo apt install wrk -y
   ```

2. **Generate Load for Web App**
   ```bash
   # Get your ALB DNS name first
   ALB_DNS="your-alb-dns-name-here"
   
   # Light load test
   ab -n 1000 -c 10 http://$ALB_DNS/
   
   # Heavy load to trigger scaling
   ab -n 10000 -c 100 http://$ALB_DNS/
   
   # Sustained load using hey
   hey -z 10m -c 50 http://$ALB_DNS/
   
   # Using wrk for sustained load
   wrk -t12 -c400 -d30s http://$ALB_DNS/
   ```

3. **Generate Load for API**
   ```bash
   # API endpoint load test
   ab -n 5000 -c 50 http://$ALB_DNS/api/data
   
   # Health check endpoint
   ab -n 1000 -c 20 http://$ALB_DNS/health
   
   # Mixed load pattern using curl
   for i in {1..100}; do
     curl http://$ALB_DNS/api/data &
     sleep 0.1
   done
   
   # Continuous API load with wrk
   wrk -t8 -c200 -d60s http://$ALB_DNS/api/data
   
   # Monitor while running load
   watch -n 2 "curl -s http://$ALB_DNS/api/data | jq ."
   ```

### F. Monitoring Scaling Events

1. **CloudWatch Dashboards**
   - Go to CloudWatch → Dashboards
   - Create "ECS Scaling Dashboard"
   - Add widgets for:
     - Service CPU/Memory utilization
     - Task count over time
     - ALB request count
     - Scaling activities

2. **Set Up Scaling Alarms**
   - **High CPU Alarm**: >90% for 5 minutes
   - **Low CPU Alarm**: <20% for 10 minutes
   - **Task Count Alarm**: When tasks = max capacity
   - **Failed Task Alarm**: When tasks fail to start

3. **View Scaling Activities**
   - ECS Console → Service → Events tab
   - Application Auto Scaling console → Scaling activities
   - CloudWatch → Logs → Auto Scaling logs

### G. Advanced Scaling Strategies

1. **Predictive Scaling (Manual Setup)**
   - Analyze historical patterns
   - Create scheduled actions based on patterns
   - Use CloudWatch Insights for pattern analysis

2. **Multi-Metric Scaling**
   - Combine CPU, Memory, and Request count
   - Create composite CloudWatch alarms
   - Use weighted scaling decisions

3. **Cross-Service Scaling**
   - Scale API service based on web app load
   - Use SQS queue depth for background services
   - Implement circuit breaker patterns

### H. Scaling Best Practices

1. **Cooldown Periods**
   - Scale-out: 300 seconds (5 minutes)
   - Scale-in: 600 seconds (10 minutes)
   - Adjust based on application startup time

2. **Health Check Grace Period**
   - Set appropriate health check grace period
   - Account for application initialization time
   - Monitor failed task launches

3. **Cost Optimization**
   - Use Fargate Spot for non-critical scaling
   - Implement aggressive scale-in policies
   - Monitor scaling costs in Cost Explorer

### I. Troubleshooting Scaling Issues

1. **Common Issues**
   - Tasks failing health checks during scale-out
   - Insufficient subnet IP addresses
   - Service limits reached
   - IAM permission issues

2. **Debugging Steps**
   - Check ECS service events
   - Review CloudWatch logs
   - Verify security group rules
   - Check subnet capacity

3. **Scaling Metrics to Monitor**
   - Average response time during scaling
   - Error rate during scale events
   - Time to scale (scale-out latency)
   - Resource utilization efficiency

## Step 12: Advanced Features (Optional)

1. **Enable Service Connect**
   - Edit service configuration
   - Enable Service Connect for service-to-service communication

2. **Blue/Green Deployments**
   - Configure CodeDeploy for ECS
   - Set up deployment configurations

## Step 13: Cleanup

1. **Delete ECS Services**
2. **Delete Task Definitions**
3. **Delete Load Balancer**
4. **Delete ECR Repositories**
5. **Delete CloudFormation Stack**

## Cost Optimization Tips

- Use Fargate Spot for non-critical workloads
- Right-size CPU/Memory allocations
- Enable Container Insights selectively
- Use scheduled scaling for predictable workloads
- Monitor and optimize using AWS Cost Explorer

## Latest ECS Features Covered

- **Fargate Platform Version 1.4+**: Latest networking and security features
- **Container Insights**: Enhanced monitoring
- **Service Connect**: Service mesh capabilities
- **ECS Exec**: Debug running containers
- **Fargate Spot**: Cost optimization
- **Blue/Green Deployments**: Zero-downtime deployments
