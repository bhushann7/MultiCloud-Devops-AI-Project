# MultiCloud, DevOps & AI Challenge - Day 3: Putting CloudApp on Autopilot with DevOps CI/CD pipeline

This document outlines the steps taken to configure and test a CI/CD pipeline for the CloudMart application using AWS CodePipeline and CodeBuild.

## Restarting Resources from Day 2

To avoid unnecessary costs, the cluster and other resources were deleted after Day 2 ended. They were then recreated at the beginning of Day 3 using the following commands:

### Create Kubernetes Cluster

```bash
eksctl create cluster \
  --name cloudmart \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 1 \
  --with-oidc \
  --managed
```

### Connect to the EKS cluster:

```bash
aws eks update-kubeconfig --name cloudmart
```

### Verify cluster connectivity:

```bash
kubectl get svc
kubectl get nodes
```

### Create a service account to provide access to AWS services:

```bash
eksctl create iamserviceaccount \
  --cluster=cloudmart \
  --name=cloudmart-pod-execution-role \
  --role-name CloudMartPodExecutionRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
  --region us-east-1 \
  --approve
```

### Switch to backend folder

```bash
cd ../..
cd challenge-day2/backend
```

### Follow the ECR steps to build your Docker image

### Create a Kubernetes deployment file (YAML) for the Backend

```bash

cd ../..
cd challenge-day2/backend
nano cloudmart-backend.yaml
```

### Content of clodmart-frontend.yaml file

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-backend-app
  template:
    metadata:
      labels:
        app: cloudmart-backend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-backend-app
        image: public.ecr.aws/l4c0j8h9/cloudmart-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-backend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-backend-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

### Deploy the Backend on Kubernetes

```bash
kubectl apply -f cloudmart-backend.yaml
```

Monitor the status of objects being created and obtain the public IP generated for the API

```bash
kubectl get pods
kubectl get deployment
kubectl get service
```

### Frontend Deployment on Kubernetes

#### Preparation

Change the Frontend's .env file to point to the API URL created within Kubernetes obtained by the kubectl get service command

```bash
cd ../challenge-day2/frontend
nano .env
```
​
Content of .env:

```bash
VITE_API_BASE_URL=http://<your_url_kubernetes_api>:5000/api
```

### Follow the ECR steps to build your Docker image

### Create a Kubernetes deployment file (YAML) for the Frontend

```bash
nano cloudmart-frontend.yaml
```
​
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-frontend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-frontend-app
  template:
    metadata:
      labels:
        app: cloudmart-frontend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-frontend-app
        image: public.ecr.aws/l4c0j8h9/cloudmart-frontend:latest
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-frontend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-frontend-app
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
```


### Deploy the Frontend on Kubernetes

```bash
kubectl apply -f cloudmart-frontend.yaml
```
​
### Monitor the status of objects being created and obtain the public IP generated for the API

```bash
kubectl get pods
kubectl get deployment
kubectl get service
```


## Part 2: CI/CD Pipeline Configuration

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
    ![Local Git Repository setup](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Local%20Git%20Repository%20setup.png)
    ![Remote Git Repository setup](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Remote%20Git%20Repository%20Setup.png)
    ![App code added to Remote Repository](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/App%20code%20added%20to%20Remote%20Repository.png)

### Step 2: Configure AWS CodePipeline

1.  Access AWS CodePipeline.
2.  Start the 'Create pipeline' process.
3.  Name: `cloudmart-cicd-pipeline`.
4.  **Important:** Select "Build custom pipeline" due to recent UI updates.
5.  Use the GitHub repository `cloudmart` as the source.
6.  **Important:** Use GitHub OAuth instead of GitHub v1 and select the appropriate repo and branch.
7.  Add the `cloudmartBuild` project as the build stage.
8.  Add the `cloudmartDeploy` project as the deployment stage.
   
   ![Create Pipeline](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Create%20Pipeline.png)
   ![Pipeline settings](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Pipeline%20setting%20p1.png)
   ![Adding source in the pipeline](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Adding%20source%20in%20the%20pipeline.png)

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
    
    ![Codebuild settings p1](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Codebuild%20settings%20p1.png)
    ![CodeBuild settings p2](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/CodeBuild%20settings%20p2.png)
    ![Buildspec file](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Buildspec%20file.png)
    ![codebuild project created](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/codebuild%20project%20created.png)
    ![Intermediate pipeline](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Created%20Pipeline.png)

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
    ![Amazon ECR Policy added to cloudmart build service role](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Amazon%20ECR%20Policy%20added%20to%20cloudmart%20build%20service%20role.png)
    ![Pipeline run succeded after policy change](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Pipeline%20run%20succeded%20after%20policy%20change.png)
    ![Adding new stage Deploy in Pipeline](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Added%20new%20stage%20Deploy.png)
    ![Deploy Stage added](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Deploy%20Stage%20added.png)

## Part 3: Test Your CI/CD Pipeline

### Step 1: Make a Change on GitHub

1.  Update application code in `cloudmart` repository.
2.  Modify file: `src/components/MainPage/index.jsx` (line 93).
3.  Commit and push:
    ```bash
    git add -A
    git commit -m "changed to Featured Products on CloudMart"
    git push
    ```
    ![Change made in source code](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Change%20made%20in%20source%20code.png)
    ![Changed pushed to remote repo](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Changed%20pushed%20to%20remote%20repo.png)

### Step 2: Observe the Pipeline Execution

1.  Watch CodePipeline automatically trigger the build.
2.  After the build completes, the deployment phase should begin.
 
    ![Git push triggered the CICD pipeline](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Git%20push%20triggered%20the%20CICD%20pipeline.png)
    ![Succesful execution of automated CICD pipeline](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Succesful%20execution%20of%20automated%20CICD%20pipeline.png)

### Step 3: Verify the Deployment

1.  Check Kubernetes using `kubectl` commands to confirm the application update.
    ```bash
    kubectl get pods
    kubectl get deployment
    kubectl get service
    ```
2.  Access the CloudMart application in a browser to verify the changes.
    
    ![Website updates after CICD run](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day3/Website%20updates%20after%20CICD%20run.png)

## Notes

1.  To avoid unnecessary costs, the cluster and other resources were deleted after Day 2. They were recreated:
2.  Remember to replace placeholder values with your actual AWS credentials and resource IDs.
3.  In a production environment, use IAM roles instead of direct credentials for CodeBuild deployment.
4.  Admin privileges were used for educational purposes; follow the principle of least privilege in production.
5.  Kubenetes clusterm,Frontend and Backend deployment was required before running this code.
6.  Verify the application is accessible at `http://<frontend_url>:5001/` after the pipeline succeeds.

