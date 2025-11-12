
# 🚀 AWS Build a Serverless AI Chatbot with Amazon Bedrock, Lambda, API Gateway & S3 🤖
<img width="1280" height="562" alt="chatbot" src="https://github.com/user-attachments/assets/035a1ea8-c743-46d5-a2ca-8ef2d73241b4" />


This hands-on demo guides you through creating a serverless AI chatbot on AWS.

---

## 🤖 Introduction
- In this step-by-step tutorial, you will create a powerful **serverless AI Chatbot** using **Amazon Bedrock’s Titan Text G1 – Express**.  
- You will connect it to **AWS Lambda**, expose it via **API Gateway** with proper **CORS**, and deploy a clean **HTML/JavaScript** frontend using **S3 static website hosting**.

---

## 🔎 Overview
👉  Users interact with the chatbot either through a static frontend hosted on **Amazon S3** or **Amplify Hosting**, or by calling the **API** directly.  
👉  Messages are sent as **HTTPS requests** to **Amazon API Gateway**, which forwards them to an **AWS Lambda** function.  
👉 The **Lambda** calls **Amazon Bedrock** (Titan Text G1 – Express) to generate responses.  
👉 The response flows back through **Lambda** and **API Gateway** to the user.

---

## 🛠 Tech Stack
- Amazon Bedrock (Titan Text G1 – Express)
- AWS Lambda
- Amazon API Gateway
- Amazon S3 (Static Hosting)
- JavaScript + HTML + CSS

---

## ➡️ Step 1 — Create an IAM Role for Lambda
Create a role for Lambda with the following permissions:
- `AmazonBedrockFullAccess`
- `CloudWatchLogsFullAccess`

Name suggestion: `chatBotLambdaRole`.

---

## ➡️ Step 2 — Create the Lambda Function
Create a Lambda function:
1. In the navigation pane search for Lambda function
2. Choose Create function.
3. Select Author from scratch.
4. In the Basic information pane, for Function name, enter chatbotlambda
5. For Runtime, choose Python 3.12 (easiest for Bedrock SDK usage).
6. Leave architecture set to x86_64, and then choose Create function.
7. For the Lambda function code, copy and paste the code below into your Lambda code editor:


---
<details>
  <summary>🧠 Lambda_function.py (click to expand)</summary>

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

def lambda_handler(event, context):
    # Handle preflight (OPTIONS) request for CORS
    if event['httpMethod'] == 'OPTIONS':
        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Headers': '*',
                'Access-Control-Allow-Methods': 'OPTIONS,POST,GET'
            },
            'body': json.dumps('Preflight OK')
        }

    # Parse incoming JSON
    body = json.loads(event['body'])
    user_message = body.get('message', '')
    history = body.get('history', [])

    # Optional: construct prompt with history
    conversation = ""
    for turn in history:
        conversation += f"User: {turn['user']}\nAssistant: {turn['assistant']}\n"
    conversation += f"User: {user_message}\nAssistant:"

    # Create payload for Titan
    request_body = {
        "inputText": conversation,
        "textGenerationConfig": {
            "maxTokenCount": 300,
            "temperature": 0.7,
            "topP": 0.9,
            "stopSequences": []
        }
    }

    # Call Titan model
    response = bedrock.invoke_model(
        modelId='amazon.titan-text-express-v1',
        body=json.dumps(request_body),
        contentType='application/json',
        accept='application/json'
    )

    # Extract model output
    result = json.loads(response['body'].read())
    reply = result.get('results', [{}])[0].get('outputText', '')

    # Return response with CORS headers
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': '*',
            'Access-Control-Allow-Methods': 'OPTIONS,POST,GET'
        },
        'body': json.dumps({'response': reply})
    }
```

</details>


---
## ➡️ Step 3 — Set Up API Gateway (REST)
1. Create a **REST API**.
2. Create a **/chat** resource.
3. Enable **CORS** on **/chat** (creates an **OPTIONS** method).
4. Create a **POST** method integrated with the Lambda function **with Lambda Proxy Integration enabled**.
5. Deploy to a stage (for example: `dev`).
6. Note the **Invoke URL**.

---

## ➡️ Step 4 — Frontend Chat UI
Create `index.html` (below).  
Replace `YOUR_API_INVOKE_URL` with your deployed API Gateway endpoint, for example:  
`https://abc123.execute-api.us-east-1.amazonaws.com/dev/chat`

---

## ➡️ Step 5 — Deploy Frontend to S3 Static Website
We'll deploy our fully serverless AI chatbot to S3 for static website hosting.

1. In the AWS Management Console, navigate to Amazon S3, click on "Create Bucket"
2. For General configuration, choose choose General purpose buckets.
3. Enter a unique bucket name, i'll name myaichatbotdemo
4. Make sure you disable "Block all public access" to have public access.
5. Keep everything else as default and click "Create bucket"
6. Upload the index.html file that you created in step 5
7. Go to "Properties" and scroll down to "Static Website Hosting" and click on "Edit"
8. Under "Static Website Hosting", choose "Enable"
9. Specify index.html as the index document, then click "Save"
10. Go to "Permissions" under Bucket Policy click "Edit"
11. Paste the Bucket Policy below, that grants read-only access to all objects (s3:GetObject) inside a specific S3 bucket

**Bucket Policy (replace `your-bucket-name`)**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    }
  ]
}
```

---
⚠️Note: Replace your-bucket-name with your actual bucket name, then click "Save"

12. Go back to the S3 Bucket console, choose Objects, then click on index.html
13. To visit your fully serverless AI chatbot Live, click on the Object URL.
14. You should see your AI Chatbot with a stylish chat interface running on Amazon S3.
🏆 Now you can ask the AI Chatbot anything and you will have a real-time AI responses.


---
##
<details>
  <summary>🕓 Downtime Summary (click to expand)</summary>

### Overview of Potential Downtime Scenarios

| Cause | Description | Mitigation |
|------|-------------|------------|
| **API Gateway misconfiguration** | Missing CORS headers or incorrect integration response can cause 4xx errors. | Verify CORS setup and enable "Lambda Proxy Integration." |
| **Lambda timeout or permissions** | IAM role missing `bedrock:InvokeModel` or `CloudWatchLogs` permissions may cause 500 errors. | Attach correct IAM policy and monitor CloudWatch logs. |
| **Bedrock throttling** | High-frequency requests can trigger throttling errors (429). | Add retry logic or backoff mechanism in the frontend. |
| **S3 static site access issues** | If “Block Public Access” is enabled or policy misconfigured, site becomes unavailable. | Ensure correct bucket policy and permissions. |
| **Region mismatch** | If Bedrock model region differs from Lambda region. | Deploy Lambda in `us-east-1` for Titan models. |

✅ **Tip:** Always monitor **CloudWatch Logs** and set up alarms for `5xx` errors or failed invocations.

</details>
