# MultiCloud, DevOps & AI Challenge - Day 4: Integrating AI Chatbot and Assistant

This document outlines the steps taken to integrate an AI Chatbot (Amazon Bedrock Agent) and an AI Assistant (OpenAI gpt-4o) into the CloudMart application, focusing on Infrastructure as Code (Terraform) and Kubernetes deployment.

## Table of Contents

1.  Creating Resources using Terraform
2.  Configuring the Amazon Bedrock Agent (AI Chatbot)
3.  OpenAI Assistant Configuration
4.  Redeploying the Backend with AI Assistants
5.  Testing the AI Assistants
6.  Learnings and Observations

## Creating Resources using Terraform

We began by setting up the necessary AWS resources using Terraform. This involved creating an IAM role and policy for the Lambda function, defining the Lambda function itself, and granting Bedrock permission to invoke it.

```bash
cd challenge-day2/backend/src/lambda
cp list_products.zip ../../../../terraform-project/
cd ../../../../terraform-project
```



## 1. Terraform Configuration

The following Terraform code was added to main.tf:

```terraform
# IAM Role for Lambda function
resource "aws_iam_role" "lambda_role" {
  name = "cloudmart_lambda_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# IAM Policy for Lambda function
resource "aws_iam_role_policy" "lambda_policy" {
  name = "cloudmart_lambda_policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:Scan",
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = [
          aws_dynamodb_table.cloudmart_products.arn,
          aws_dynamodb_table.cloudmart_orders.arn,
          aws_dynamodb_table.cloudmart_tickets.arn,
          "arn:aws:logs:*:*:*"
        ]
      }
    ]
  })
}

# Lambda function for listing products
resource "aws_lambda_function" "list_products" {
  filename         = "list_products.zip"
  function_name    = "cloudmart-list-products"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("list_products.zip")

  environment {
    variables = {
      PRODUCTS_TABLE = aws_dynamodb_table.cloudmart_products.name
    }
  }
}

# Lambda permission for Bedrock
resource "aws_lambda_permission" "allow_bedrock" {
  statement_id  = "AllowBedrockInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.list_products.function_name
  principal     = "bedrock.amazonaws.com"
}

# Output the ARN of the Lambda function
output "list_products_function_arn" {
  value = aws_lambda_function.list_products.arn
}
```

## 2. Configuring the Amazon Bedrock Agent (AI Chatbot)

This section details the steps to create and configure the Amazon Bedrock Agent for CloudMart, enabling product recommendations.

### Model Access:

-   Navigate to the Amazon Bedrock console and access the Model Catalog.
-   Select Claude 3 Sonnet and modify access to enable it.
-   Wait for the access to be granted.

![Amazon Bedrock Model Catalog](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Amazon%20Bedrock%20Model%20Catalog.png)
![Request to Access granted]()

### Creating the Agent:

-   Create a new agent named "cloudmart-product-recommendation-agent".
-   Select Claude 3 Sonnet as the base model.
-   Provide the agent instructions for product recommendations.

![Create Agent](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Create%20Agent.png)
![Selecting model for agent](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Selecting%20model%20for%20agent.png)
![Agent created](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Agent%20Created.png)

### Agent Instructions:
```bash
You are a product recommendations agent for CloudMart, an online e-commerce store. Your role is to assist customers in finding products that best suit their needs. Follow these instructions carefully:

1. Begin each interaction by retrieving the full list of products from the API. This will inform you of the available products and their details.

2. Your goal is to help users find suitable products based on their requirements. Ask questions to understand their needs and preferences if they're not clear from the user's initial input.

3. Use the 'name' parameter to filter products when appropriate. Do not use or mention any other filter parameters that are not part of the API.

4. Always base your product suggestions solely on the information returned by the API. Never recommend or mention products that are not in the API response.

5. When suggesting products, provide the name, description, and price as returned by the API. Do not invent or modify any product details.

6. If the user's request doesn't match any available products, politely inform them that we don't currently have such products and offer alternatives from the available list.

7. Be conversational and friendly, but focus on helping the user find suitable products efficiently.

8. Do not mention the API, database, or any technical aspects of how you retrieve the information. Present yourself as a knowledgeable sales assistant.

9. If you're unsure about a product's availability or details, always check with the API rather than making assumptions.

10. If the user asks about product features or comparisons, use only the information provided in the product descriptions from the API.

11. Be prepared to assist with a wide range of product inquiries, as our e-commerce store may carry various types of items.

12. If a user is looking for a specific type of product, use the 'name' parameter to search for relevant items, but be aware that this may not capture all categories or types of products.

Remember, your primary goal is to help users find the best products for their needs from what's available in our store. Be helpful, informative, and always base your recommendations on the actual product data provided by the API.
```
### Configuring the IAM Role:

- In the Bedrock Agent overview, locate the 'Permissions' section.
- Click on the IAM role link. This will take you to the IAM console with the correct role selected.
- In the IAM console, choose "Add permissions" and then "Create inline policy".
- In the JSON tab, paste the following policy:


### IAM Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:*:*:function:cloudmart-list-products"
    },
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0"
    }
  ]
}
```
![Update Policy for Agent IAM role]()

### Configuring the Action Group:

-   Create an action group named "Get-Product-Recommendations".
-    Set the action group type as "Define with API schemas".
- Select the Lambda function "cloudmart-list-products" as the action group executor.
-   Define the API schema using the inline schema editor.


### OpenAPI Schema:

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "Product Details API",
    "version": "1.0.0",
    "description": "This API retrieves product information..."
  },
  "paths": {
    "/products": {
      "get": {
        "summary": "Retrieve product details",
        "description": "Retrieves a list of products...",
        "parameters": [
          {
            "name": "name",
            "in": "query",
            "description": "Retrieve details...",
            "schema": {
              "type": "string"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Successful response",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "properties": {
                      "name": {
                        "type": "string"
                      },
                      "description": {
                        "type": "string"
                      },
                      "price": {
                        "type": "number"
                      }
                    }
                  }
                }
              }
            }
          },
          "500": {
            "description": "Internal Server Error",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/ErrorResponse"
                }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "ErrorResponse": {
        "type": "object",
        "properties": {
          "error": {
            "type": "string",
            "description": "Error message"
          }
        },
        "required": ["error"]
      }
    }
  }
}
```


![Create Action Group](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Create%20Action%20Group.png)
![Action Group Created](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Action%20Group%20Created.png)


### Review and Create:

-   Review all configurations and prepare the agent.

### Testing the Agent:

-   Test the agent using the "Test Agent" panel.
-   Verify if the agent asks relevant questions and provides accurate product recommendations.

### Creating an Alias:

-   Create an alias named "cloudmart-prod" for the agent.

![Alias created](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Alias%20Created.png)

## 3. OpenAI Assistant Configuration

This section covers the setup of the OpenAI gpt-4o assistant for customer support.

### OpenAI Access:

-   Access the OpenAI platform and create an account.
-   Add credit to your account.

### Creating the Assistant:

-   Create a new assistant named "CloudMart Customer Support".
-   Select gpt-4o as the model.

![OpenAI assistant created](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/OpenAI%20assistant%20created.png)

### Configuring the Assistant:

-   Provide instructions for customer support.

### Assistant Instructions:

```bash
You are a customer support agent for CloudMart, an e-commerce platform. Your role is to assist customers with general inquiries, order issues, and provide helpful information about using the CloudMart platform. You don't have direct access to specific product or inventory information. Always be polite, patient, and focus on providing excellent customer service. If a customer asks about specific products or inventory, politely explain that you don't have access to that information and suggest they check the website or speak with a sales representative.
```

- In "Capabilities", you can enable "Code Interpreter" if you want the assistant to help with technical aspects of using the platform.

### Saving the Assistant:

- The assistant auto-saves
-   Note down the Assistant ID.

![Testing the AI agent](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Testing%20the%20AI%20agent.png)

### Generating API Key:

-   Generate and copy the API key.

## 4. Redeploying the Backend with AI Assistants

To integrate the AI assistants into the backend, we updated the `cloudmart-backend.yaml` file with the necessary environment variables.

**Updating `cloudmart-backend.yaml`:**

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

### Replacing Placeholders:

- Replace the xxxx placeholders with the actual BEDROCK_AGENT_ID, BEDROCK_AGENT_ALIAS_ID, OPENAI_API_KEY, and OPENAI_ASSISTANT_ID.

### Applying the Deployment:

```bash
kubectl apply -f cloudmart-backend.yaml
```

![Updated Bedrock and OpenAI keys in Kubernetes](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Updated%20Bedrock%20and%20OpenAI%20keys%20in%20kubenetes%20deployment%20file.png)

## 5. Testing the AI Assistants
After redeploying the backend, we tested both the AI Chatbot and the AI Assistant.


### AI Chatbot (Amazon Bedrock Agent):

- We interacted with the chatbot to request product recommendations.
- Verified that the chatbot asked relevant questions and provided accurate product suggestions based on the API data.
- [Screenshot opportunity: Add a screenshot of the AI Chatbot interaction showing a successful product recommendation.]

![AI Assistant chatbot working](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/AI%20Assistant%20chatbot%20working.png)
![Engaging with chatbot](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Engaging%20with%20chatbot.png)
![Chatbot describing the product by taking description from backend](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Chatbot%20describing%20the%20product%20by%20taking%20description%20from%20backend.png)
![Interaction with chatbot](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Interaction%20with%20chatbot.png)
![Interaction with chatbot 2](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Interaction%20with%20chatbot%202.png)

### Assistant (OpenAI gpt-4o):

- We tested the assistant with general inquiries and order-related tasks, such as requesting order cancellation.
- Verified that the assistant provided helpful and polite responses.
- [Screenshot opportunity: Add a screenshot of the AI Assistant interaction showing a successful order cancellation request.]
- [Screenshot opportunity: Add a screenshot of the order history showing the cancelled order.]

![Order Placed successfully](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Order%20Placed%20successfully.png)
![Chat with agent to cancel the order](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Chat%20with%20agent%20to%20cancel%20the%20order.png)
![Order cancelled by customer support bot](https://github.com/bhushann7/MultiCloud-Devops-AI-Project/blob/main/Screenshots/Day4/Order%20cancelled%20by%20customer%20support%20bot.png)

## 6. Learnings and Observations
### Amazon Bedrock Agent Configuration:
- The process of configuring the Bedrock Agent required careful attention to IAM roles, policies, and API schemas.
- The agent's ability to retrieve and present product information accurately depended heavily on the quality of the API schema and instructions.

### OpenAI Assistant Integration:
- Integrating the OpenAI assistant was straightforward, requiring only the API key and assistant ID.
- The assistant's performance was impressive, providing helpful and relevant responses.
