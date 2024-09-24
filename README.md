# webapp-containers

Here’s a step-by-step guide, along with sample code, for deploying a web application using containers. In this example, we’ll use **Docker** for containerization and **Amazon ECS with Fargate** for deployment. We will also push the Docker image to **Amazon ECR (Elastic Container Registry)**.

### **1. Containerizing the Web Application with Docker**

#### Step 1: Create a Simple Web Application
Create a simple Node.js or Python web application as an example.

**Node.js Example** (`app.js`):
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello World from Docker!');
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`App running on port ${port}`);
});
```

#### Step 2: Create a Dockerfile
This file defines the environment and steps to build the Docker image.

```Dockerfile
# Use an official Node.js runtime as a parent image
FROM node:14

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json to install dependencies
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the application code to the container
COPY . .

# Expose the port your app runs on
EXPOSE 3000

# Command to run the application
CMD [ "node", "app.js" ]
```

#### Step 3: Build the Docker Image
Build the Docker image using the following command:

```bash
docker build -t my-web-app .
```

#### Step 4: Test the Application Locally
Run the Docker container to verify everything works:

```bash
docker run -p 3000:3000 my-web-app
```

### **2. Push the Docker Image to Amazon ECR**

#### Step 1: Create a Repository in ECR
Create a new ECR repository using the AWS Management Console or CLI:

```bash
aws ecr create-repository --repository-name my-web-app
```

#### Step 2: Authenticate Docker to ECR
Run the following command to authenticate Docker to your ECR registry (replace `<aws-region>` with your region):

```bash
aws ecr get-login-password --region <aws-region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<aws-region>.amazonaws.com
```

#### Step 3: Tag and Push the Docker Image
Tag your Docker image to match the ECR repository URI:

```bash
docker tag my-web-app:latest <account-id>.dkr.ecr.<aws-region>.amazonaws.com/my-web-app:latest
```

Push the image to ECR:

```bash
docker push <account-id>.dkr.ecr.<aws-region>.amazonaws.com/my-web-app:latest
```

### **3. Set Up ECS with Fargate for Deployment**

#### Step 1: Create an ECS Cluster
Create an ECS cluster that uses Fargate as the launch type:

```bash
aws ecs create-cluster --cluster-name my-ecs-cluster
```

#### Step 2: Create a Task Definition
Create an ECS task definition that specifies the Docker container image, memory, CPU, and networking details.

Sample ECS Task Definition (`task-def.json`):
```json
{
  "family": "my-web-app-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "my-web-app",
      "image": "<account-id>.dkr.ecr.<aws-region>.amazonaws.com/my-web-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "essential": true
    }
  ],
  "requiresCompatibilities": [ "FARGATE" ],
  "cpu": "256",
  "memory": "512"
}
```

Register the task definition:

```bash
aws ecs register-task-definition --cli-input-json file://task-def.json
```

#### Step 3: Create an ECS Service
Create an ECS service to run the task continuously. This service will run on the ECS cluster using Fargate.

```bash
aws ecs create-service \
    --cluster my-ecs-cluster \
    --service-name my-web-app-service \
    --task-definition my-web-app-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-xyz],securityGroups=[sg-xyz],assignPublicIp=ENABLED}"
```

Make sure to replace `subnet-xyz` and `sg-xyz` with the appropriate subnet and security group IDs in your AWS VPC.

### **4. Set Up Load Balancer (Optional)**
To expose your application publicly, you might want to set up an **Application Load Balancer** (ALB). Here’s how:

#### Step 1: Create an ALB
Create a load balancer in the same VPC and subnets as your ECS service.

#### Step 2: Create a Target Group
Create a target group for your ECS tasks.

```bash
aws elbv2 create-target-group \
    --name my-web-app-targets \
    --protocol HTTP \
    --port 3000 \
    --vpc-id <vpc-id>
```

#### Step 3: Attach ALB to ECS Service
Update your ECS service to register tasks with the target group:

```bash
aws ecs update-service \
    --cluster my-ecs-cluster \
    --service my-web-app-service \
    --load-balancers targetGroupArn=<target-group-arn>,containerName=my-web-app,containerPort=3000
```

### **5. Monitor and Auto-Scale**

#### Step 1: Set Up CloudWatch Alarms
Set up CloudWatch alarms to monitor CPU or memory usage. When an alarm is triggered, you can scale the number of ECS tasks automatically.

#### Step 2: Set Up Auto Scaling
Enable ECS Service Auto Scaling by creating scaling policies that trigger based on CloudWatch metrics (like CPU utilization).

```bash
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/my-ecs-cluster/my-web-app-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 1 \
    --max-capacity 5

aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --scalable-dimension ecs:service:DesiredCount \
    --resource-id service/my-ecs-cluster/my-web-app-service \
    --policy-name my-scale-out-policy \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration file://scaling-policy.json
```

### **6. Continuous Deployment (Optional)**
To automate future deployments, set up a CI/CD pipeline using tools like **AWS CodePipeline** or **Jenkins** to automatically push new versions of the Docker image to ECR and update your ECS service.

---

This workflow allows you to deploy a web application using containers, push the image to ECR, and run the app on ECS using Fargate. You can also integrate auto-scaling and monitoring through CloudWatch.

Let me know if you need additional guidance on any specific step!
