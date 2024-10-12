# Node.js Application CI/CD Pipeline with GitHub Actions, ECR, and ECS (Fargate)

This project demonstrates how to set up a CI/CD pipeline for a Node.js application using GitHub Actions, AWS Elastic Container Registry (ECR), and AWS Elastic Container Service (ECS) with Fargate.

## Project Overview

The Node.js application is containerized using Docker, pushed to AWS ECR, and deployed to ECS (Fargate). The pipeline is triggered on changes pushed to the `main` branch, automatically handling the build, push, and deployment processes.

---

## Prerequisites

1. **AWS Account**: Make sure you have an AWS account with the necessary permissions to interact with ECR, ECS, and Fargate.
2. **AWS CLI**: Install and configure the AWS CLI.
   ```bash
   aws configure
   ```
3. **GitHub Repository**: A GitHub repository containing the Node.js application along with a `Dockerfile`.
4. **IAM User**: Create an AWS IAM user with the required permissions and add the credentials (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) as GitHub secrets.
5. **ECR and ECS**:
   - Create an **ECR repository** to store Docker images.
   - Create an **ECS cluster** with **Fargate** as the launch type.
   - Define the **Task Definition** in a JSON file (`task_definition.json`) for the ECS service.

---

## Steps to Set Up CI/CD Pipeline

### 1. AWS Resource Configuration

1. **Create an ECR Repository**:
   - Go to AWS Console > ECR > Create a repository (e.g., `node-ecs-cicd`).
   
   Example ECR URI: `123456789012.dkr.ecr.ap-southeast-2.amazonaws.com/node-ecs-cicd`

2. **Create an ECS Cluster**:
   - Go to AWS Console > ECS > Create a cluster (using Fargate).
   
3. **Create a Task Definition**:
   - Define a task definition JSON (`task_definition.json`) for your container, specifying the container settings, image, memory, CPU, etc.

4. **Create an ECS Service**:
   - Create a service in the ECS cluster, associating it with the task definition and Fargate.

---

### 2. GitHub Actions Setup

1. **GitHub Secrets**:
   - Go to your GitHub repository > Settings > Secrets and Variables > Actions > New repository secret.
   - Add the following secrets:
     - `AWS_ACCESS_KEY_ID`: Your AWS IAM access key.
     - `AWS_SECRET_ACCESS_KEY`: Your AWS IAM secret access key.
   
2. **GitHub Actions Workflow**:
   Create a `.github/workflows/deploy.yml` file in your repository with the following content:

   ```yaml
   name: Deploy to Amazon ECS

   on:
     push:
       branches:
         - main

   env:
     AWS_REGION: ap-southeast-2
     ECR_REPOSITORY: node-ecs-cicd
     ECS_SERVICE: node-ecs-cicd-service
     ECS_CLUSTER: node-ecs-cicd
     ECS_TASK_DEFINITION: .aws/task_definition.json
     CONTAINER_NAME: "node-ecs-cicd"

   jobs:
     deploy:
       name: Deploy
       runs-on: ubuntu-latest
       environment: development

       steps:
         - name: Checkout
           uses: actions/checkout@v3

         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v1
           with:
             aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
             aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
             aws-region: ${{ env.AWS_REGION }}

         - name: Login to Amazon ECR
           id: login-ecr
           uses: aws-actions/amazon-ecr-login@v1

         - name: Build, tag, and push image to Amazon ECR
           id: build-image
           env:
             ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
             IMAGE_TAG: ${{ github.sha }}
           run: |
             docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
             docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
             echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

         - name: Fill in the new image ID in the Amazon ECS task definition
           id: task-def
           uses: aws-actions/amazon-ecs-render-task-definition@v1
           with:
             task-definition: ${{ env.ECS_TASK_DEFINITION }}
             container-name: ${{ env.CONTAINER_NAME }}
             image: ${{ steps.build-image.outputs.image }}

         - name: Deploy Amazon ECS task definition
           uses: aws-actions/amazon-ecs-deploy-task-definition@v1
           with:
             task-definition: ${{ steps.task-def.outputs.task-definition }}
             service: ${{ env.ECS_SERVICE }}
             cluster: ${{ env.ECS_CLUSTER }}
             wait-for-service-stability: true
   ```

---

### 3. Testing and Deployment

1. **Push Changes to GitHub**:
   - Push your code changes to the `main` branch. This triggers the pipeline.
   
2. **Monitor the Workflow**:
   - Go to the **Actions** tab in GitHub to monitor the progress of the pipeline.
   
3. **Verify Deployment**:
   - Check the ECS service in AWS to confirm that the latest version of the application is deployed.
   - Access the application through the ECS service's public IP or load balancer.

---
