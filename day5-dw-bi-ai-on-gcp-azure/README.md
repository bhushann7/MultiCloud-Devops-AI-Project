# MultiCloud, DevOps & AI Challenge - Day 5: Going Multicloud: DW/BI on Google Cloud & Sentiment Analysis with Microsoft Azure AI

## Overview

This document outlines the steps taken to integrate **Google BigQuery** for order data backup and **Azure Text Analytics** for sentiment analysis into the **CloudMart** application. The setup focuses on **Infrastructure as Code (Terraform)** and **Kubernetes deployment** to create a scalable, automated cloud solution.

## Table of Contents

1.  Download Updated Frontend and Backend Code
2.  Google Cloud BigQuery Setup
3.  Terraform Setup for BigQuery
4.  Azure Text Analytics Setup
5.  Deploy Backend Changes
6.  Validate Setup
7.  Learnings and Observations

## Step 1: Download Updated Frontend and Backend Code

Before making any updates, ensure you back up the existing folders:

```bash
cp -R cloudmart-web-app/ cloudmart-web-app_bkp
cp -R terraform-project/ terraform-project_bkp
```

### Clean Up Existing Files

Remove unnecessary files while keeping Docker and YAML files intact.

#### Backend Cleanup:

```bash
cd cloudmart-web-app/backend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

#### Frontend Cleanup:

```bash
cd cloudmart-web-app/frontend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

### Copy the Updated Code

#### Backend:

```bash
cd cloudmart-web-app/backend
cp -r website-cloud-devops-ai/cloudmart-backend-final/* cloudmart-web-app/backend
```

#### Frontend:

```bash
cd cloudmart-web-app/frontend
cp -r website-cloud-devops-ai/cloudmart-frontend-final/* cloudmart-web-app/frontend
```

### Commit and Push Changes

```bash
git add -A
git commit -m "Final code update"
git push
```

## Step 2: Google Cloud BigQuery Setup

### 1. Create a Google Cloud Project

-   Navigate to [Google Cloud Console](https://console.cloud.google.com/).
-   Create a new project named **CloudMart**.

### 2. Enable BigQuery API

-   Go to **APIs & Services** > **Dashboard**.
-   Enable the **BigQuery API**.

### 3. Create a BigQuery Dataset

-   Navigate to **BigQuery** in the Google Cloud Console.
-   Create a dataset named `cloudmart`.

### 4. Create a BigQuery Table

-   In the dataset `cloudmart`, create a table named `cloudmart-orders` with the following schema:
    -   `id: STRING`
    -   `createdAt: TIMESTAMP`
    -   `items: JSON`
    -   `status: STRING`
    -   `total: FLOAT`
    -   `userEmail: STRING`



### 5. Create Service Account and Key

-   Navigate to **IAM & Admin** > **Service Accounts**.
-   Create a service account `cloudmart-bigquery-sa` with **BigQuery Data Editor** role.
-   Generate a JSON key and save it as `google_credentials.json`.

## Step 3: Terraform Setup for BigQuery

-   Remove existing `main.tf` and create a new one:

```bash
rm main.tf
nano main.tf
```

-   Define AWS, DynamoDB, and Lambda configurations in `main.tf`. Ensure Lambda is triggered by DynamoDB stream to back up data in BigQuery.

-   Apply the changes:


```bash
terraform apply
```

## Step 4: Azure Text Analytics Setup

### 1. Create an Azure Account

-   Sign in or create an account at [Azure Portal](https://portal.azure.com/).

### 2. Create a Language Service Resource

-   Search for **Language Service** and create a new resource.

### 3. Retrieve API Credentials

-   Copy the **Endpoint URL** and **API Key** from the resource settings.

## Step 5: Deploy Backend Changes

### Update Deployment Configuration

Modify `cloudmart-backend.yaml` with required environment variables:

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
        image: public.ecr.aws/l4c0j8h9/cloudmaster-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
        - name: AZURE_ENDPOINT
          value: "xxxx"
        - name: AZURE_API_KEY
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
- Replace the key-value pairs for credentials with the appropriate values

### Deploy Changes

```bash
kubectl apply -f cloudmart-backend.yaml
```

## Step 6: Validate Setup

### BigQuery Backup

-   Place a test order and verify if the data appears in **Google BigQuery**.

### Azure Sentiment Analysis

-   Submit a support ticket and check for sentiment analysis results.


## Learnings and Observations

### Google BigQuery Integration

-   Setting up BigQuery required configuring service accounts and IAM roles correctly to ensure smooth data ingestion.

-   Data schema design significantly impacts query performance and efficiency.


### Azure Text Analytics

-   API response accuracy depends on the quality of input text; well-structured customer queries yield better sentiment analysis results.

-   Using Azure’s pre-trained AI models simplifies natural language processing tasks without requiring custom ML training.

This integration enhances **CloudMart's** data processing capabilities, ensuring **reliability, scalability, and real-time insights** across multiple cloud environments. 🚀


## Summary

This project successfully integrates **Google BigQuery** for automated order data backup and **Azure Text Analytics** for sentiment analysis, creating a robust multicloud setup. By leveraging **Terraform**, **Kubernetes**, and cloud-native services, the deployment ensures scalability and automation. The key steps included:

1.  **Code Preparation & Cleanup** – Backing up existing files, updating backend/frontend, and committing changes to Git.
2.  **Google Cloud BigQuery Integration** – Setting up datasets, tables, and service accounts to store order history.
3.  **Terraform Configuration** – Automating cloud resource creation and ensuring seamless data flow.
4.  **Azure Language Analytics** – Enabling sentiment analysis for support interactions.
5.  **Backend Deployment** – Updating configurations and deploying changes to Kubernetes.
 6.  **Validation & Testing** – Verifying successful order backups in **BigQuery** and sentiment analysis results from **Azure**.