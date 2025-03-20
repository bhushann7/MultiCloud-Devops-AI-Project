# MultiCloud, DevOps & AI Challenge - Day 3: CI/CD Pipeline for CloudMart

This document outlines the steps taken to configure and test a CI/CD pipeline for the CloudMart application using AWS CodePipeline and CodeBuild.

## Part 1: CI/CD Pipeline Configuration

### Step 1: Create GitHub Repository

1.  Create a free account on GitHub.
2.  Create a new repository called `cloudmart`.
3.  Navigate to the frontend directory:
    ```bash
    cd challenge-day2/frontend
    ```
4.  Initialize and push the code to GitHub:
    ```bash
    git status
    git add -A
    git commit -m "app sent to repo"
    git push
    ```
    **(Screenshot: Successful push to GitHub repository)**

### Step 2: Configure AWS CodePipeline

1.  Access AWS CodePipeline.
2.  Start the 'Create pipeline' process.
3.  Name: `cloudmart-cicd-pipeline`.
4.  **Important:** Select "Build custom pipeline" due to recent UI updates.
5.  Use the GitHub repository `cloudmart` as the source.
6.  **Important:** Use GitHub OAuth instead of GitHub v1 and select the appropriate repo and branch.
7.  Add the `cloudmartBuild` project as the build stage.
8.  Add the `cloudmartDeploy` project as the deployment stage.
    **(Screenshot: AWS CodePipeline configuration)**

### Step 3: Configure AWS CodeBuild to Build the Docker Image

1.  Create a Build Project:
    * Name: `cloudmartBuild`.
    * Connect to GitHub repository: `cloudmart`.
    * Image: `amazonlinux2-x86_64-standard:4.0`.
    * Enable Docker builds: Check "Enable this flag if you want to build Docker images or want your builds to get elevated privileges".
    * Add environment variable:
        * Name: `ECR_REPO`.
        * Value: [Your ECR repository URI].
    * Use the following `buildspec.yml`:
        ```yaml
        version: 0.2
        phases:
          install:
            runtime-versions:
              docker: 20
          pre_build:
            commands:
              - echo Logging in to Amazon ECR...
              - aws --version
              - REPOSITORY_URI=$ECR_REPO
              - aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/l4c0j8h9
          build:
            commands:
              - echo Build started on `date`
              - echo Building the Docker image...
              - docker build -t $REPOSITORY_URI:latest .
              - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
          post_build:
            commands:
              - echo Build completed on `date`
              - echo Pushing the Docker image...
              - docker push $REPOSITORY_URI:latest
              - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
              - export imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION
              - printf '[{\"name\":\"cloudmart-app\",\"imageUri\":\"%s\"}]' $REPOSITORY_URI:$imageTag > imagedefinitions.json
              - cat imagedefinitions.json
              - ls -l
        env:
          exported-variables: ["imageTag"]
        artifacts:
          files:
            - imagedefinitions.json
            - cloudmart-frontend.yaml
        ```
    * Add Permissions:
        * Access IAM console > Roles.
        * Find role created for CodeBuild (`cloudmartBuild`).
        * Add permission: `AmazonElasticContainerRegistryPublicFullAccess`.
    **(Screenshot: AWS CodeBuild configuration for build stage)**
    **(Screenshot: IAM role with added ECR permissions)**

### Step 4: Configure AWS CodeBuild for Application Deployment

1.  Create a Deployment Project:
    * Name: `cloudmartDeployToProduction`.
    * Configure environment variables:
        * `AWS_ACCESS_KEY_ID`: [eks-user access key].
        * `AWS_SECRET_ACCESS_KEY`: [eks-user secret key].
        * Note: In production, use an IAM role instead of direct credentials (refer to "Enabling IAM principal access to your cluster").
    * Use the following `buildspec.yml`:
        ```yaml
        version: 0.2
        phases:
          install:
            runtime-versions:
              docker: 20
            commands:
              - curl -o kubectl [https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd6004/kubectl](https://www.google.com/search?q=https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd6004/kubectl)
              - chmod +x ./kubectl
              - mv ./kubectl /usr/local/bin
              - kubectl version --short --client
          post_build:
            commands:
              - aws eks update-kubeconfig --region us-east-1 --name cloudmart
              - kubectl get nodes
              - ls
              - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
              - echo $IMAGE_URI
              - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" cloudmart-frontend.yaml
              - kubectl apply -f cloudmart-frontend.yaml
        ```
    * In `cloudmart-frontend.yaml`:
        * Replace the image URI on line 18 with `CONTAINER_IMAGE`.
        * Commit and push changes:
            ```bash
            git add -A
            git commit -m "replaced image uri with CONTAINER_IMAGE"
            git push
            ```
    **(Screenshot: AWS CodeBuild configuration for deployment stage)**
    **(Screenshot: Updated cloudmart-frontend.yaml in GitHub)**

## Part 2: Test Your CI/CD Pipeline

### Step 1: Make a Change on GitHub

1.  Update application code in `cloudmart` repository.
2.  Modify file: `src/components/MainPage/index.jsx` (line 93).
3.  Commit and push:
    ```bash
    git add -A
    git commit -m "changed to Featured Products on CloudMart"
    git push
    ```
    **(Screenshot: Updated index.jsx file in GitHub)**

### Step 2: Observe the Pipeline Execution

1.  Watch CodePipeline automatically trigger the build.
2.  After the build completes, the deployment phase should begin.
    **(Screenshot: AWS CodePipeline execution in progress)**
    **(Screenshot: AWS CodePipeline successful execution)**

### Step 3: Verify the Deployment

1.  Check Kubernetes using `kubectl` commands to confirm the application update.
    ```bash
    kubectl get pods
    kubectl get deployment
    kubectl get service
    ```
    **(Screenshot: kubectl get pods, deployment, service output)**
2.  Access the CloudMart application in a browser to verify the changes.
    **(Screenshot: Updated CloudMart application in browser)**

## Notes

1.  To avoid unnecessary costs, the cluster and other resources were deleted after Day 2. They were recreated using the following commands:
    ```bash
    eksctl create cluster \
      --name cloudmart \
      --region us-east-1 \
      --nodegroup-name standard-workers \
      --node-type t3.medium \
      --nodes 1 \
      --with-oidc \
      --managed

    aws eks update-kubeconfig --name cloudmart

    kubectl get svc
    kubectl get nodes

    eksctl create iamserviceaccount \
      --cluster=cloudmart \
      --name=cloudmart-pod-execution-role \
      --role-name CloudMartPodExecutionRole \
      --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess \
      --region us-east-1 \
      --approve
    ```
2.  Remember to replace placeholder values with your actual AWS credentials and resource IDs.
3.  In a production environment, use IAM roles instead of direct credentials for CodeBuild deployment.
4.  Admin privileges were used for educational purposes; follow the principle of least privilege in production.
5.  Backend deployment was required before running this code. The commands used are as follows:

    ```bash
    cd ../..
    cd challenge-day2/backend
    kubectl apply -f cloudmart-backend.yaml
    kubectl get pods
    kubectl get deployment
    kubectl get service
    ```
6.  Frontend deployment was required before running this code. The commands used are as follows:

    ```bash
    cd ../challenge-day2/frontend
    nano .env
    #Update .env with backend url
    kubectl apply -f cloudmart-frontend.yaml
    kubectl get pods
    kubectl get deployment
    kubectl get service
    ```
7.  Verify the application is accessible at `http://<frontend_url>:5001/` after the pipeline succeeds.
**(Screenshot: CloudMart application at http://<frontend_url>:5001/)**
**(Screenshot: CloudMart admin panel at http://<frontend_url>:5001/admin)**