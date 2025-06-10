CircleCI Complete Tutorial: From Setup to Production Deployment
Table of Contents
What is CircleCI
Creating Your CircleCI Account
Connecting GitHub Repository
Understanding the Config File
Writing Your First Pipeline
Branch Strategy and Workflows
Approval Workflows
Environment Variables and Secrets
Docker Integration
AWS EKS Deployment
Troubleshooting Common Issues
What is CircleCI
CircleCI is a continuous integration and continuous deployment (CI/CD) platform that automates the software development process. It helps you:

Test code automatically when you push changes
Build and deploy applications to various environments
Maintain code quality through automated checks
Speed up development with parallel processing
Ensure consistency across different environments
Key Benefits
Cloud-based (no server maintenance)
Easy GitHub/Bitbucket integration
Parallel job execution
Docker support
AWS, Azure, GCP integration
Free tier available
Creating Your CircleCI Account
Step 1: Sign Up
Go to https://circleci.com
Click "Sign Up"
Choose "Sign up with GitHub" (recommended) or "Sign up with Bitbucket"
Authorize CircleCI to access your repositories
Step 2: Account Setup
Choose your plan: Start with the free tier (1,000 build minutes/month)
Select organization: Choose your personal account or organization
Complete profile: Add your name and company information
Step 3: Verify Setup
You should see your dashboard with no projects yet
Your GitHub repositories will be listed for connection
Connecting GitHub Repository
Step 1: Add Project
From the CircleCI dashboard, click "Projects"
Find your repository in the list
Click "Set Up Project" next to your repository
Step 2: Choose Configuration Method
CircleCI will ask how you want to configure your project:

Option A: Use existing config (if you already have .circleci/config.yml)

Click "Use Existing Config"
CircleCI will use your existing configuration
Option B: Create new config

Choose your language/framework (Node.js for our project)
CircleCI will generate a basic config file
You can customize it later
Step 3: First Build
Click "Start Building"
CircleCI will run your first build
Monitor the build process in real-time
Repository Structure
Your repository should have this structure:

your-repo/
├── .circleci/
│   └── config.yml
├── src/
├── package.json
└── README.md
Understanding the Config File
The .circleci/config.yml file is the heart of your CI/CD pipeline. Here's a breakdown:

Basic Structure
version: 2.1
orbs:          # Pre-built packages of CircleCI configuration
executors:     # Execution environments  
jobs:          # Individual units of work
workflows:     # Orchestrates job execution
Our Complete Configuration Explained
Version and Orbs
version: 2.1
orbs:
  node: circleci/node@5.1.0      # Node.js commands and environments
  docker: circleci/docker@2.2.0  # Docker build and push commands
  aws-cli: circleci/aws-cli@4.0.0 # AWS CLI installation and setup
Orbs are reusable packages that contain jobs, commands, and executors. They save time by providing pre-configured functionality.

Executors
executors:
  node-executor:
    docker:
      - image: cimg/node:20.9.0   # Node.js 20.9.0 environment
  aws-executor:
    docker:
      - image: cimg/aws:2023.09   # AWS CLI environment
Executors define the execution environment where your jobs run.

Jobs
Each job performs specific tasks:

Test Job:

test:
  executor: node-executor
  steps:
    - checkout                    # Download code from repository
    - node/install-packages:      # Install npm dependencies
        pkg-manager: npm
    - run:
        name: Run tests
        command: ./test.sh        # Execute test script
    - store_test_results:         # Save test results
        path: test-results
Build and Push Job:

build-and-push:
  executor: node-executor
  steps:
    - checkout
    - setup_remote_docker:        # Enable Docker commands
        version: 20.10.14
        docker_layer_caching: true
    - docker/check:               # Login to Docker Hub
        docker-username: DOCKER_USERNAME
        docker-password: DOCKER_PASSWORD
    - docker/build:               # Build Docker image
        image: hilltopconsultancy/devops-hilltop
        tag: << pipeline.git.revision >>
    - docker/push:                # Push to Docker Hub
        image: hilltopconsultancy/devops-hilltop
        tag: << pipeline.git.revision >>
Deploy Job:

deploy-staging:
  executor: aws-executor
  steps:
    - checkout
    - aws-cli/setup:              # Configure AWS credentials
        aws-access-key-id: AWS_ACCESS_KEY_ID
        aws-secret-access-key: AWS_SECRET_ACCESS_KEY
        aws-region: AWS_DEFAULT_REGION
    - run:
        name: Configure kubectl for EKS
        command: |
          aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $EKS_CLUSTER_NAME
    - run:
        name: Deploy to EKS staging
        command: |
          kubectl apply -f k8s/
          kubectl rollout status deployment/devops-hilltop-deployment -n devops-hilltop --timeout=300s
Writing Your First Pipeline
Step 1: Create Basic Config
Start with a simple configuration:

version: 2.1
jobs:
  hello-world:
    docker:
      - image: cimg/base:stable
    steps:
      - run:
          name: Say Hello
          command: echo "Hello, World!"
workflows:
  my-workflow:
    jobs:
      - hello-world
Step 2: Add Node.js Testing
version: 2.1
orbs:
  node: circleci/node@5.1.0
jobs:
  test:
    executor: node/default
    steps:
      - checkout
      - node/install-packages
      - run:
          name: Run tests
          command: npm test
workflows:
  test-and-build:
    jobs:
      - test
Step 3: Add Build Stage
version: 2.1
orbs:
  node: circleci/node@5.1.0
jobs:
  test:
    executor: node/default
    steps:
      - checkout
      - node/install-packages
      - run: npm test
  build:
    executor: node/default
    steps:
      - checkout
      - node/install-packages
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist
workflows:
  test-and-build:
    jobs:
      - test
      - build:
          requires:
            - test
Step 4: Add Conditional Logic
workflows:
  test-and-deploy:
    jobs:
      - test
      - build:
          requires:
            - test
      - deploy-staging:
          requires:
            - build
          filters:
            branches:
              only: develop    # Only deploy develop branch
      - deploy-production:
          requires:
            - build
          filters:
            branches:
              only: main       # Only deploy main branch
Branch Strategy and Workflows
Git Flow with CircleCI
Branch Strategy:

main branch      → Production deployment (manual approval)
develop branch   → Staging deployment (automatic)
feature branches → Tests only (no deployment)
Workflow Configuration
workflows:
  version: 2
  build-test-deploy:
    jobs:
      # Run tests on all branches
      - test
      # Build and push only on main/develop
      - build-and-push:
          requires:
            - test
          filters:
            branches:
              only:
                - main
                - develop
      # Auto-deploy to staging from develop
      - deploy-staging:
          requires:
            - build-and-push
          filters:
            branches:
              only: develop
      # Manual approval for production
      - hold-for-approval:
          type: approval
          requires:
            - build-and-push
          filters:
            branches:
              only: main
      # Deploy to production after approval
      - deploy-production:
          requires:
            - hold-for-approval
          filters:
            branches:
              only: main
How It Works
Feature Branch Workflow:

Create feature branch from develop
Push changes → Triggers tests only
Create Pull Request to develop
Merge after review
Staging Deployment:

Push to develop branch
Tests run automatically
Build Docker image
Deploy to staging environment
No manual intervention required
Production Deployment:

Merge develop to main
Tests and build run
Manual approval required
Deploy to production after approval
Approval Workflows
Setting Up Manual Approvals
Configuration:

workflows:
  production-deploy:
    jobs:
      - test
      - build:
          requires:
            - test
      - hold-for-approval:
          type: approval
          requires:
            - build
          filters:
            branches:
              only: main
      - deploy-production:
          requires:
            - hold-for-approval
Approval Process
Step 1: Automatic Trigger

Push to main branch
Tests and build complete successfully
Pipeline pauses at approval step
Step 2: Manual Review

Go to CircleCI dashboard
Find the paused workflow
Review build artifacts and test results
Click "Approve" or "Cancel"
Step 3: Deployment

After approval, production deployment starts
Monitor deployment progress
Verify successful deployment
Best Practices for Approvals
Who Should Approve:

Team leads or senior developers
DevOps engineers
Product managers for major releases
What to Check:

All tests pass
Build artifacts are correct
No security vulnerabilities
Database migrations are safe
Rollback plan is ready
Approval Settings:

hold-for-approval:
  type: approval
  requires:
    - build
  filters:
    branches:
      only: main
Environment Variables and Secrets
Types of Variables
1. Project Environment Variables

Stored in CircleCI project settings
Available to all jobs in the project
Used for API keys, passwords, configuration
2. Context Variables

Shared across multiple projects
Managed at organization level
Used for common credentials
3. Built-in Variables

Provided by CircleCI automatically
Examples: CIRCLE_BRANCH, CIRCLE_SHA1, CIRCLE_BUILD_NUM
Setting Up Environment Variables
Step 1: Access Project Settings

Go to your project in CircleCI
Click "Project Settings"
Select "Environment Variables"
Step 2: Add Variables Click "Add Environment Variable" and add these:

Docker Hub Variables:

Name: DOCKER_USERNAME
Value: hilltopconsultancy
Name: DOCKER_PASSWORD
Value: your_docker_hub_access_token
AWS Variables:

Name: AWS_ACCESS_KEY_ID
Value: your_aws_access_key
Name: AWS_SECRET_ACCESS_KEY
Value: your_aws_secret_key
Name: AWS_DEFAULT_REGION
Value: eu-central-1
Name: EKS_CLUSTER_NAME
Value: devops-hilltop-cluster
Using Variables in Config
In Commands:

- run:
    name: Deploy to cluster
    command: |
      aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $EKS_CLUSTER_NAME
In Orb Parameters:

- aws-cli/setup:
    aws-access-key-id: AWS_ACCESS_KEY_ID
    aws-secret-access-key: AWS_SECRET_ACCESS_KEY
    aws-region: AWS_DEFAULT_REGION
Dynamic Values:

- docker/build:
    image: hilltopconsultancy/devops-hilltop
    tag: << pipeline.git.revision >>  # Git commit SHA
Security Best Practices
Never hardcode secrets:

# ❌ Bad
- run: docker login -u hilltopconsultancy -p mypassword
# ✅ Good
- docker/check:
    docker-username: DOCKER_USERNAME
    docker-password: DOCKER_PASSWORD
Use contexts for shared secrets:

workflows:
  deploy:
    jobs:
      - deploy-production:
          context: aws-production  # Shared AWS credentials
Docker Integration
Docker Hub Setup
Step 1: Create Docker Hub Account

Go to hub.docker.com
Sign up with username: hilltopconsultancy
Create repository: devops-hilltop
Step 2: Generate Access Token

Go to Account Settings → Security
Click "New Access Token"
Name: CircleCI-DevOps-Hilltop
Permissions: Read, Write, Delete
Copy the generated token
Step 3: Configure CircleCI Add these environment variables:

DOCKER_USERNAME: hilltopconsultancy
DOCKER_PASSWORD: your_access_token_here
Docker Configuration in CircleCI
Basic Docker Build:

version: 2.1
orbs:
  docker: circleci/docker@2.2.0
jobs:
  build-and-push:
    executor: docker/docker
    steps:
      - checkout
      - setup_remote_docker
      - docker/build:
          image: hilltopconsultancy/devops-hilltop
          tag: latest
      - docker/push:
          image: hilltopconsultancy/devops-hilltop
          tag: latest
Advanced Docker Build:

build-and-push:
  executor: docker/docker
  steps:
    - checkout
    - setup_remote_docker:
        version: 20.10.14
        docker_layer_caching: true  # Cache layers for faster builds
    - docker/check:
        docker-username: DOCKER_USERNAME
        docker-password: DOCKER_PASSWORD
    - docker/build:
        image: hilltopconsultancy/devops-hilltop
        tag: << pipeline.git.revision >>  # Use git commit as tag
    - docker/push:
        image: hilltopconsultancy/devops-hilltop
        tag: << pipeline.git.revision >>
    - docker/build:
        image: hilltopconsultancy/devops-hilltop
        tag: latest
    - docker/push:
        image: hilltopconsultancy/devops-hilltop
        tag: latest
Dockerfile Best Practices
Multi-stage Build:

# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
Security Optimizations:

FROM node:20-alpine
# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
# Set working directory
WORKDIR /app
# Copy and install dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
# Copy application code
COPY . .
# Change ownership to non-root user
RUN chown -R nextjs:nodejs /app
USER nextjs
EXPOSE 5000
CMD ["npm", "start"]
AWS EKS Deployment
AWS Setup Prerequisites
Step 1: AWS Account Setup

Create AWS account
Create IAM user for CircleCI
Attach necessary policies:
AmazonEKSClusterPolicy
AmazonEKSWorkerNodePolicy
AmazonEC2ContainerRegistryPowerUser
Step 2: Create EKS Cluster

eksctl create cluster \
  --name devops-hilltop-cluster \
  --region eu-central-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed
Step 3: Configure kubectl

aws eks update-kubeconfig --region eu-central-1 --name devops-hilltop-cluster
CircleCI AWS Integration
AWS CLI Orb Configuration:

orbs:
  aws-cli: circleci/aws-cli@4.0.0
jobs:
  deploy-to-eks:
    executor: aws-cli/default
    steps:
      - checkout
      - aws-cli/setup:
          aws-access-key-id: AWS_ACCESS_KEY_ID
          aws-secret-access-key: AWS_SECRET_ACCESS_KEY
          aws-region: AWS_DEFAULT_REGION
      - run:
          name: Configure kubectl
          command: |
            aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $EKS_CLUSTER_NAME
      - run:
          name: Deploy to EKS
          command: |
            kubectl apply -f k8s/
            kubectl rollout status deployment/devops-hilltop-deployment -n devops-hilltop
Kubernetes Manifests
Namespace:

apiVersion: v1
kind: Namespace
metadata:
  name: devops-hilltop
Deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-hilltop-deployment
  namespace: devops-hilltop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: devops-platform
  template:
    metadata:
      labels:
        app: devops-platform
    spec:
      containers:
      - name: devops-hilltop
        image: hilltopconsultancy/devops-hilltop:latest
        ports:
        - containerPort: 5000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: devops-hilltop-secret
              key: DATABASE_URL
Service (NodePort):

apiVersion: v1
kind: Service
metadata:
  name: devops-hilltop-service
  namespace: devops-hilltop
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30080
  selector:
    app: devops-platform
Database Deployment
PostgreSQL Deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  namespace: devops-hilltop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_DB
          value: "devops_hilltop"
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
Deployment Process
Complete Deployment Pipeline:

deploy-staging:
  executor: aws-executor
  steps:
    - checkout
    - aws-cli/setup:
        aws-access-key-id: AWS_ACCESS_KEY_ID
        aws-secret-access-key: AWS_SECRET_ACCESS_KEY
        aws-region: AWS_DEFAULT_REGION
    - run:
        name: Configure kubectl for EKS
        command: |
          aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $EKS_CLUSTER_NAME
    - run:
        name: Update image tag
        command: |
          sed -i "s|image: hilltopconsultancy/devops-hilltop:latest|image: hilltopconsultancy/devops-hilltop:<< pipeline.git.revision >>|g" k8s/deployment.yaml
    - run:
        name: Deploy to EKS
        command: |
          kubectl apply -f k8s/
          kubectl rollout status deployment/postgres-deployment -n devops-hilltop --timeout=300s
          kubectl rollout status deployment/devops-hilltop-deployment -n devops-hilltop --timeout=300s
    - run:
        name: Initialize database
        command: |
          kubectl exec deployment/devops-hilltop-deployment -n devops-hilltop -- npm run db:push
    - run:
        name: Verify deployment
        command: |
          kubectl get pods -n devops-hilltop
          kubectl get services -n devops-hilltop
          NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
          NODE_PORT=$(kubectl get service devops-hilltop-service -n devops-hilltop -o jsonpath='{.spec.ports[0].nodePort}')
          echo "Application URL: http://$NODE_IP:$NODE_PORT"
Troubleshooting Common Issues
Build Failures
1. Test Failures

Error: Tests failed with exit code 1
Solution:

Check test logs in CircleCI
Run tests locally: npm test
Fix failing tests before pushing
2. Docker Build Failures

Error: failed to solve with frontend dockerfile.v0
Solution:

Verify Dockerfile syntax
Check if base image exists
Ensure all COPY paths are correct
3. Docker Push Failures

Error: unauthorized: authentication required
Solution:

Verify DOCKER_USERNAME and DOCKER_PASSWORD
Check Docker Hub repository exists
Regenerate Docker Hub access token
Deployment Issues
1. kubectl Authentication

Error: You must be logged in to the server (Unauthorized)
Solution:

Verify AWS credentials
Check EKS cluster name and region
Ensure IAM user has EKS permissions
2. Pod Startup Failures

Error: ImagePullBackOff
Solution:

Verify Docker image exists and is public
Check image tag in deployment manifest
Ensure Kubernetes can access Docker Hub
3. Database Connection Issues

Error: ECONNREFUSED
Solution:

Verify PostgreSQL pod is running
Check database credentials in secrets
Ensure service names match in configuration
Environment Variable Issues
1. Missing Variables

Error: Environment variable AWS_ACCESS_KEY_ID not set
Solution:

Add missing variables in project settings
Check variable names for typos
Verify context is attached to job
2. Invalid Credentials

Error: The AWS Access Key Id you provided does not exist
Solution:

Regenerate AWS access keys
Update environment variables
Check IAM user permissions
Workflow Issues
1. Jobs Not Running

Error: No workflows to run for this pipeline
Solution:

Check workflow syntax in config.yml
Verify job names match in requires section
Ensure filters are correctly configured
2. Approval Not Working

Error: Workflow stuck at approval step
Solution:

Check if user has approval permissions
Verify approval job configuration
Manually approve in CircleCI dashboard
Performance Issues
1. Slow Builds

Enable Docker layer caching
Use smaller base images
Parallelize independent jobs
Cache dependencies between builds
2. Resource Limits

Error: Resource limit exceeded
Solution:

Upgrade CircleCI plan
Optimize resource usage
Use resource classes efficiently
Debug Techniques
1. SSH into Build Environment

- run:
    name: Setup SSH
    command: |
      echo "Add SSH key in CircleCI project settings"
      # Click 'Rerun job with SSH' in CircleCI
2. Add Debug Logging

- run:
    name: Debug Environment
    command: |
      echo "Current directory: $(pwd)"
      echo "Environment variables:"
      env | grep -E '^(AWS|DOCKER)' | sort
      echo "Kubernetes config:"
      kubectl config current-context
3. Save Debug Artifacts

- store_artifacts:
    path: build-logs
    destination: debug-logs
Getting Help
CircleCI Resources:

CircleCI Documentation
CircleCI Discuss
CircleCI Status Page
Community Support:

Stack Overflow (tag: circleci)
GitHub Issues for orbs
DevOps community forums
Professional Support:

CircleCI Support (paid plans)
CircleCI Professional Services
DevOps consultants
Summary
You now have a complete understanding of CircleCI from basic setup to production deployment. Here's what you've learned:

✅ Account Setup and Repository Connection ✅ Config File Structure and Syntax ✅ Writing Effective CI/CD Pipelines ✅ Branch Strategy and Workflow Management ✅ Manual Approval Processes ✅ Environment Variables and Security ✅ Docker Integration and Registry Management ✅ AWS EKS Deployment Automation ✅ Troubleshooting and Debugging

Next Steps
Set up your CircleCI account and connect your repository
Configure environment variables for Docker Hub and AWS
Test the pipeline by pushing to develop branch
Monitor deployments and set up notifications
Optimize performance with caching and parallelization
Add monitoring and alerting for production deployments
Your DevOps with Hilltop application is now ready for professional CI/CD with CircleCI!
