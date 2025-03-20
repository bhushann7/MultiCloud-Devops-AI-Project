# MultiCloud, DevOps & AI Challenge - Day 2: Deploying the app with Docker & Kubernetes Deployment on AWS

This document outlines the steps taken to deploy the CloudMart application using Docker and Kubernetes on AWS Elastic Kubernetes Service (EKS).

## Part 1: Docker

### Step 1: Install Docker on EC2

1.  **Update system packages:**
    ```bash
    sudo yum update -y
    ```
2.  **Install Docker:**
    ```bash
    sudo yum install docker -y
    ```
3.  **Start Docker service:**
    ```bash
    sudo systemctl start docker
    ```
4.  **Verify Docker installation:**
    ```bash
    sudo docker run hello-world
    ```
    **(Screenshot: Output of `docker run hello-world`)**
5.  **Enable Docker to start on boot:**
    ```bash
    sudo systemctl enable docker
    ```
6.  **Check Docker version:**
    ```bash
    docker --version
    ```
    **(Screenshot: Output of `docker --version`)**
7.  **Add user to the Docker group:**
    ```bash
    sudo usermod -a -G docker $(whoami)
    newgrp docker
    ```

### Step 2: Create Docker Image for CloudMart

1.  **Create project directory and navigate to backend:**
    ```bash
    mkdir -p challenge-day2/backend && cd challenge-day2/backend
    ```
2.  **Download and extract backend source code:**
    ```bash
    wget [https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip](https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip)
    unzip cloudmart-backend.zip
    ```

    ![Backend code unzipped in repo](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Unzip%20backend%20code%20in%20backend%20directory.png)

3.  **Create `.env` file:**
    ```bash
    nano .env
    ```
    **Content of `.env`:**
    ```
    PORT=5000
    AWS_REGION=us-east-1
    BEDROCK_AGENT_ID=<your-bedrock-agent-id>
    BEDROCK_AGENT_ALIAS_ID=<your-bedrock-agent-alias-id>
    OPENAI_API_KEY=<your-openai-api-key>
    OPENAI_ASSISTANT_ID=<your-openai-assistant-id>
    ```
    **Note:** Replace placeholders with your actual AWS Bedrock and OpenAI credentials.
4.  **Create `Dockerfile` for backend:**
    ```bash
    nano Dockerfile
    ```
    **Content of `Dockerfile`:**
    ```dockerfile
    FROM node:18
    WORKDIR /usr/src/app
    COPY package*.json ./
    RUN npm install
    COPY . .
    EXPOSE 5000
    CMD ["npm", "start"]
    ```
5.  **Navigate to the parent directory and create frontend directory:**
    ```bash
    cd ..
    mkdir frontend && cd frontend
    ```
6.  **Download and extract frontend source code:**
    ```bash
    wget [https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip](https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip)
    unzip cloudmart-frontend.zip
    ```

    ![Frontend code unzipped in frontend repo](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Unzip%20frontend%20code%20in%20frontendn%20directory.png)

7.  **Create `Dockerfile` for frontend:**
    ```bash
    nano Dockerfile
    ```
    **Content of `Dockerfile`:**
    ```dockerfile
    FROM node:16-alpine as build
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci
    COPY . .
    RUN npm run build

    FROM node:16-alpine
    WORKDIR /app
    RUN npm install -g serve
    COPY --from=build /app/dist /app
    ENV PORT=5001
    ENV NODE_ENV=production
    EXPOSE 5001
    CMD ["serve", "-s", ".", "-l", "5001"]
    ```
 ![]()
 ![Dockerfile for Frontend](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Dockerfile%20for%20Frontend.png)

## Part 2: Kubernetes

**Important:** AWS EKS incurs charges. Remember to delete the cluster after use.

### Cluster Setup on AWS EKS


![eksuser with admin priveleges](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/eksuser%20with%20Admin%20privieleges.png)
![Setting Access Key for eksuser](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Setting%20Access%20Key%20for%20eksuser.png)

1.  **Configure AWS CLI:**
    ```bash
    aws configure
    ```
2.  **Install `eksctl`:**
    ```bash
    curl --silent --location "[https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$]([https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$)(uname]([https://www.google.com/search?q=https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_%24)(uname](https://www.google.com/search?q=https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_%24)(uname)) -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo cp /tmp/eksctl /usr/bin
    eksctl version
    ```
    
3.  **Install `kubectl`:**
    ```bash
    curl -o kubectl [https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl](https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl)
    chmod +x ./kubectl
    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
    echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
    kubectl version --short --client
    ```

    ![eksctl and kubectl installed](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Succesful%20installation%20of%20eksctl%20and%20kubectl.png)

4.  **Create EKS cluster:**
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

    ![Kubernetes cluster creation in progress](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Kubernetes%20cluster%20creation%20in%20process.png)

    * **Check cluster creation status:** You can monitor the cluster creation process on the AWS CloudFormation console: [https://console.aws.amazon.com/cloudformation/](https://console.aws.amazon.com/cloudformation/)


    ![Kubernetes cluster creation in progress in CloudFormation](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Kubernetes%20cluster%20creation%20progress%20on%20CloudFormation.png)

5.  **Update `kubectl` configuration:**
    ```bash
    aws eks update-kubeconfig --name cloudmart
    ```
6.  **Verify cluster connectivity:**
    ```bash
    kubectl get svc
    kubectl get nodes
    ```
   
   ![Kubernetes cluster creation complete in CloudFormation](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Cluster%20creation%20complete%20on%20CloudFormation.png)

7.  **Create IAM service account:**
    ```bash
    eksctl create iamserviceaccount \
      --cluster=cloudmart \
      --name=cloudmart-pod-execution-role \
      --role-name CloudMartPodExecutionRole \
      --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess \
      --region us-east-1 \
      --approve
    ```


### Backend Deployment

1.  **Create ECR repository for backend:**
    * Name: `cloudmart-backend`
2.  **Build and push Docker image to ECR:**
    * Follow ECR instructions from AWS console.
    
![backend ECR Repo](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Create%20an%20ECR%20Repository%20for%20the%20Backend.png)

3.  **Create `cloudmart-backend.yaml`:**
    ```bash
    cd ../..
    cd challenge-day2/backend
    nano cloudmart-backend.yaml
    ```
    **Content of `cloudmart-backend.yaml`:**
    ```yaml
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
            image: <your-ecr-backend-image-uri>:latest
            env:
            - name: PORT
              value: "5000"
            - name: AWS_REGION
              value: "us-east-1"
            - name: BEDROCK_AGENT_ID
              value: "xxxxxx"
            - name: BEDROCK_AGENT_ALIAS_ID
            - value: "xxxx"
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
    ![backend image uri](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Backend%20Image%20URI%20for%20Kubernetes%20deployment%20file.png)
    **Note:** Replace `<your-ecr-backend-image-uri>` and credential placeholders.
4.  **Deploy backend:**
    ```bash
    kubectl apply -f cloudmart-backend.yaml
    ```
5.  **Monitor deployment and get service IP:**
    ```bash
    kubectl get pods
    kubectl get deployment
    kubectl get service
    ```

![backend image deployed](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Backend%20Deployed%20on%20Kubernetes.png)

### Frontend Deployment

1.  **Update frontend `.env` with backend service IP:**
    ```bash
    cd ../challenge-day2/frontend
    nano .env
    ```
    **Content of `.env`:**
    ```
    VITE_API_BASE_URL=http://<your_backend_service_ip>:5000/api
    ```
    **Note:** Replace `<your_backend_service_ip>` with the external IP obtained from `kubectl get service`.
2.  **Create ECR repository for frontend:**
    * Name: `cloudmart-frontend`
3.  **Build and push Docker image to ECR:**
    * Follow ECR instructions from AWS console.
    
![frontend ECR Repo](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Create%20an%20ECR%20Repository%20for%20the%20Frontend.png)

4.  **Create `cloudmart-frontend.yaml`:**
    ```bash
    nano cloudmart-frontend.yaml
    ```
    **Content of `cloudmart-frontend.yaml`:**
    ```yaml
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
            image: <your-ecr-frontend-image-uri>:latest
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
    **Note:** Replace `<your-ecr-frontend-image-uri>`.
5.  **Deploy frontend:**
    ```bash
    kubectl apply -f cloudmart-frontend.yaml
    ```
6.  **Monitor deployment and get service IP:**
    ```bash
    kubectl get pods
    kubectl get deployment
    kubectl get service
    ```
![frontend and backend deployed](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Both%20Frontend%20and%20backend%20deployed%20%20on%20Kubernetes.png)

7.  **Access the application:**
    * Open `http://<your_frontend_service_ip>:5001/` in a web browser.
    
    ![website deployed successfully](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Succesfully%20deployed%20the%20website%20.jpeg)

    * Admin panel: `http://<your_frontend_service_ip>:5001/admin`
    
    ![Adding new products](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Adding%20New%20product%20using%20admin%20panel.png)

    ![new products added](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/New%20Products%20Added.png)



### Verification

* **DynamoDB:** Verify that added products and orders are reflected in the `cloudmart-products` and `cloudmart-orders` tables, respectively. Also, check the `cloudmart-tickets` table for customer service conversations.
    
![DynamoDB rows](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day2/Products%20added%20in%20DynamoDB.png)    

### Removal

1.  **Delete Kubernetes services and deployments:**
    ```bash
    kubectl delete service cloudmart-frontend-app-service
    kubectl delete deployment cloudmart-frontend-app
    kubectl delete service cloudmart-backend-app-service
    kubectl delete deployment cloudmart-backend-app
    ```
2.  **Delete EKS cluster:**
    ```bash
    eksctl delete cluster --name cloudmart --region us-east-1
    ```

### Notes

* AWS EKS incurs charges. Remember to delete the cluster after use.
* Application accessible at `http://<frontend_url>:5001/`.
* Ensure to replace all placeholder values with your actual AWS credentials and resource IDs.
* Admin privileges were used for educational purposes; follow the principle of least privilege in production.