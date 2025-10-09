# 🚀 Automated Docker Image Deployment to Amazon ECR with Jenkins and Lambda Integration

## 🧩 Overview
This project demonstrates a complete **CI/CD pipeline** for Dockerized applications using **Jenkins**, **Amazon ECR**, **EventBridge**, **Lambda**, **DynamoDB**, and **SNS**.

You can set it up in two ways:
- **Manual Setup (No Terraform / No CLI)**
- **Automated Setup using Terraform**

---

## 🎯 Objective
Automate the workflow where every code push:

- 🏗️ Builds a Docker image in Jenkins  
- 📦 Pushes it to Amazon ECR  
- ⚡ Triggers AWS Lambda via EventBridge  
- 🗃️ Logs image details in DynamoDB  
- 📧 Sends notifications through SNS  

---

## 🧱 Architecture Diagram

```mermaid
sequenceDiagram
  participant Dev
  participant GitHub
  participant Jenkins
  participant ECR
  participant EventBridge
  participant Lambda
  participant DynamoDB
  participant SNS

  Dev->>GitHub: Push code
  GitHub->>Jenkins: Webhook trigger
  Jenkins->>Docker: Build image
  Jenkins->>ECR: Push image
  ECR->>EventBridge: Image Push Event
  EventBridge->>Lambda: Invoke
  Lambda->>DynamoDB: Log image data
  Lambda->>SNS: Notify success
```
---
## 🧩 PART 1: Manual AWS Setup (No Terraform)

### 🧠 Flow Summary

| Step | Action | Description |
|------|---------|-------------|
| 1️⃣ | Create ECR repository | Store Docker images |
| 2️⃣ | Prepare application code | Node.js + Dockerfile |
| 3️⃣ | Configure Jenkins | Build & push Docker image |
| 4️⃣ | Create Lambda function | Log and notify |
| 5️⃣ | Create DynamoDB + SNS | Store & send notifications |
| 6️⃣ | Create EventBridge rule | Trigger Lambda on push |
| 7️⃣ | Trigger & verify | Confirm CI/CD flow |
---
### 1️⃣ Go to AWS Console → ECR → Create repository

  1. Name: sample-app-repo

  2. Tag mutability: Mutable

  3. Enable scan on push (optional)

  4. Copy repository URI
     </br>
     
  👉 Example:
  `
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/sample-app-repo
  `

---

### 2️⃣ Prepare Application Code

#### Folder Structure:
```pgsql
auto-ecr-jenkins/
├─ app/
│  ├─ Dockerfile
│  ├─ package.json
│  └─ index.js
└─ jenkins/
   └─ Jenkinsfile
```

#### app/index.js

```js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;
app.get('/', (req, res) => res.json({ message: 'Hello from Dockerized app!' }));
app.listen(port, () => console.log(`Running on port ${port}`));
```


#### app/package.json

```json
{
  "name": "sample-app",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": { "start": "node index.js" },
  "dependencies": { "express": "^4.18.2" }
}
```

#### app/Dockerfile

```Dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

---


### 3️⃣ Configure Jenkins



#### 🔹 Install Plugins

  - Docker plugin

  - Pipeline plugin

  - GitHub integration

  - Amazon ECR plugin (optional)



#### 🔹 Add Credentials

  - Type: Username with password

  - ID: `aws-creds-id`

  - Username:<YOUR AWS_ACCESS_KEY_ID>

  - Password:<YOUR AWS_SECRET_ACCESS_KEY>




#### 🔹 Jenkinsfile

##### jenkins/Jenkinsfile

```groovy
pipeline {
  agent any

  environment {
    AWS_REGION = 'ap-south-1'
    AWS_ACCOUNT = '123456789012'
    ECR_REPO_NAME = 'sample-app-repo'
    ECR_URL = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.IMAGE_TAG = new Date().format("yyyyMMdd-HHmmss") + "-" + sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        dir('app') {
          sh "docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Login to ECR & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds-id', usernameVariable: 'AWS_ID', passwordVariable: 'AWS_SECRET')]) {
          sh '''
            export AWS_ACCESS_KEY_ID=$AWS_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET
            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
            docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_URL}:${IMAGE_TAG}
            docker push ${ECR_URL}:${IMAGE_TAG}
            aws events put-events --entries "[{\\"Source\\":\\"custom.jenkins\\",\\"DetailType\\":\\"ECR Image Push\\",\\"Detail\\":\\"{\\\\\\"repository\\\\\\": \\\\\\"${ECR_REPO_NAME}\\\\\\", \\\\\\"imageTag\\\\\\": \\\\\\"${IMAGE_TAG}\\\\\\"}\\"}]"
          '''
        }
      }
    }
  }

  post {
    success {
      echo " Docker image pushed to ECR: ${ECR_URL}:${IMAGE_TAG}"
    }
    failure {
      echo " Build failed"
    }
  }
}
```
---

### 4️⃣ Lambda Function

- Runtime: Python 3.10
  
- Name: `ecr-postprocessor`

#### lambda/ecr_postprocessor.py

```python
import json, os, boto3
from datetime import datetime

ddb = boto3.client('dynamodb')
sns = boto3.client('sns')

def lambda_handler(event, context):
    detail = event.get('detail', {})
    repo = detail.get('repository', 'unknown')
    tag = detail.get('imageTag', 'unknown')
    ts = datetime.utcnow().isoformat()

    ddb.put_item(
        TableName=os.environ['DDB_TABLE'],
        Item={
            'imageTag': {'S': tag},
            'repository': {'S': repo},
            'timestamp': {'S': ts}
        }
    )

    sns.publish(
        TopicArn=os.environ['SNS_ARN'],
        Message=f"Image pushed: {repo}:{tag} at {ts}",
        Subject="ECR Image Push Notification"
    )
    return {'status': 'ok'}
```

#### Environment Variables:

- DDB_TABLE = `sample-app-image-log`

- SNS_ARN = `arn:aws:sns:ap-south-1:123456789012:sample-app-topic`

### 5️⃣ DynamoDB & SNS

- DynamoDB Table:

    + Name: `sample-app-image-log`

    + Partition key: `imageTag (String)`

- SNS Topic:

    + Name: `sample-app-topic`

    + Create email subscription and confirm via inbox

### 6️⃣ EventBridge Rule

---

  + Event Pattern:
```json
{
  "source": ["custom.jenkins"],
  "detail-type": ["ECR Image Push"]
}
```

  + Target → your Lambda function (`ecr-postprocessor`)

---

### 7️⃣ Verify Flow


✅ Push code to GitHub </br>
✅ Jenkins builds & pushes image → ECR</br>
✅ Jenkins triggers EventBridge → Lambda</br>
✅ Lambda logs record → DynamoDB</br>
✅ SNS sends email → success 🎉</br>


---

🧩 PART 2: Terraform Setup (Infrastructure as Code)
📁 Folder Structure
terraform/
├─ main.tf
├─ variables.tf
├─ outputs.tf
└─ lambda/
   └─ ecr_postprocessor.py

⚙️ main.tf
provider "aws" {
  region = var.region
}

# 1️⃣ ECR Repository
resource "aws_ecr_repository" "sample_repo" {
  name = var.ecr_repo_name
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration {
    scan_on_push = false
  }
}

# 2️⃣ DynamoDB Table
resource "aws_dynamodb_table" "image_log" {
  name         = var.ddb_table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "imageTag"

  attribute {
    name = "imageTag"
    type = "S"
  }
}

# 3️⃣ SNS Topic
resource "aws_sns_topic" "image_push_topic" {
  name = var.sns_topic_name
}

resource "aws_sns_topic_subscription" "email_sub" {
  topic_arn = aws_sns_topic.image_push_topic.arn
  protocol  = "email"
  endpoint  = var.notification_email
}

# 4️⃣ Lambda Function
resource "aws_iam_role" "lambda_exec_role" {
  name = "lambda_ecr_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic_exec" {
  role       = aws_iam_role.lambda_exec_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_ddb_sns_policy" {
  name = "lambda-ddb-sns-policy"
  role = aws_iam_role.lambda_exec_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "sns:Publish"
        ]
        Resource = [
          aws_dynamodb_table.image_log.arn,
          aws_sns_topic.image_push_topic.arn
        ]
      }
    ]
  })
}

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/lambda/ecr_postprocessor.py"
  output_path = "${path.module}/lambda/ecr_postprocessor.zip"
}

resource "aws_lambda_function" "ecr_postprocessor" {
  function_name = var.lambda_name
  depends_on    = [data.archive_file.lambda_zip]
  role          = aws_iam_role.lambda_exec_role.arn
  runtime       = "python3.10"
  handler       = "ecr_postprocessor.lambda_handler"
  filename      = data.archive_file.lambda_zip.output_path
  timeout       = 10

  environment {
    variables = {
      DDB_TABLE = aws_dynamodb_table.image_log.name
      SNS_ARN   = aws_sns_topic.image_push_topic.arn
    }
  }
}

# 5️⃣ EventBridge Rule + Target
resource "aws_cloudwatch_event_rule" "jenkins_push_rule" {
  name        = "JenkinsECRPushRule"
  description = "Triggers Lambda on Jenkins ECR Image Push"
  event_pattern = jsonencode({
    "source"      : ["custom.jenkins"],
    "detail-type" : ["ECR Image Push"]
  })
}

resource "aws_cloudwatch_event_target" "lambda_target" {
  rule      = aws_cloudwatch_event_rule.jenkins_push_rule.name
  target_id = "LambdaTrigger"
  arn       = aws_lambda_function.ecr_postprocessor.arn
}

resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ecr_postprocessor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.jenkins_push_rule.arn
}

⚙️ outputs.tf
output "ecr_repo_url" {
  value = aws_ecr_repository.sample_repo.repository_url
}

output "lambda_function_name" {
  value = aws_lambda_function.ecr_postprocessor.function_name
}

output "sns_topic_arn" {
  value = aws_sns_topic.image_push_topic.arn
}

⚙️ lambda/ecr_postprocessor.py
import json, os, boto3
from datetime import datetime

ddb = boto3.client('dynamodb')
sns = boto3.client('sns')

def lambda_handler(event, context):
    print("Event received:", json.dumps(event))
    detail = event.get('detail', {})
    repo = detail.get('repository', 'unknown')
    tag = detail.get('imageTag', 'unknown')
    ts = datetime.utcnow().isoformat()

    ddb.put_item(
        TableName=os.environ['DDB_TABLE'],
        Item={
            'imageTag': {'S': tag},
            'repository': {'S': repo},
            'timestamp': {'S': ts}
        }
    )

    sns.publish(
        TopicArn=os.environ['SNS_ARN'],
        Message=f"Image pushed: {repo}:{tag} at {ts}",
        Subject="ECR Image Push Notification"
    )

    return {'status': 'ok'}

⚙️ variables.tf
variable "region" { default = "ap-southeast-2" }
variable "ecr_repo_name" { default = "sample-app-repo" }
variable "ddb_table_name" { default = "sample-app-image-log" }
variable "sns_topic_name" { default = "sample-app-topic" }
variable "lambda_name" { default = "ecr-postprocessor" }
variable "notification_email" { default = "you@example.com" }

⚙️ Deploy Terraform
cd terraform
terraform init
terraform apply -auto-approve


Outputs:

ECR Repository URI

Lambda Function Name

SNS Topic ARN

Use these values in your Jenkinsfile.

🧠 Optional: Jenkins Terraform Stage
stage('Terraform Deploy') {
  steps {
    dir('terraform') {
      sh 'terraform init'
      sh 'terraform apply -auto-approve'
    }
  }
}

✅ Benefits
Feature	Benefit
Reproducible	All AWS infra in code
Scalable	Works across regions & environments
Safe	Version control + rollback
Automated	No manual AWS console setup
📜 Summary
Component	Function
GitHub	Stores source code
Jenkins	Builds Docker images & triggers events
Docker	Containerizes the app
ECR	Stores built images
EventBridge	Listens for image push events
Lambda	Logs & notifies
DynamoDB	Stores metadata
SNS	Sends notifications
🧾 Deliverables
File	Description
Dockerfile, index.js, package.json	App files
Jenkinsfile	CI/CD automation
lambda/ecr_postprocessor.py	Lambda code
terraform/*.tf	Infrastructure as code
README.md	Documentation
