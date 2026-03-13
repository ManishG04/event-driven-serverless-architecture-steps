# Event Driven Serverless Architecture Steps

## 🗝️ Step 1: The KMS Master Key (The Vault Door)
Before you build a single service, you need the Customer Managed Key (CMK) that will encrypt everything.

Go to KMS ➔ Customer managed keys ➔ Create key.

Choose Symmetric and Encrypt and decrypt.

Key Administrators: Select your own user/role so you can manage it.

Key Usage Permissions: Select the IAM Execution Role you plan to use for your Lambda function (so Lambda can decrypt SQS and encrypt DynamoDB).

🚨 THE CRITICAL TRAP: AWS Services (like S3 and SNS) do not have IAM roles you can select in the console checkboxes. After the key is created, you must click on the Key Policy, click Edit, and manually add this JSON block so S3 can encrypt SNS messages, and SNS can encrypt SQS messages:

```JSON
{
    "Sid": "Allow AWS Services to use the CMK",
    "Effect": "Allow",
    "Principal": {
        "Service": [
            "s3.amazonaws.com",
            "sns.amazonaws.com"
        ]
    },
    "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
    ],
    "Resource": "*"
}
```
## 🗄️ Step 2: DynamoDB (The Destination)
Build the end of the pipeline first.

Go to DynamoDB ➔ Create table.

Set the Partition key (e.g., MessageID).

Under Table settings, choose Customize settings.

Scroll down to Encryption at rest. Select Owned by you (Customer managed key) and pick the CMK you just created.

## 📥 Step 3: SQS Queue (The Shock Absorber)
Now build the queue that will hold the messages.

Go to SQS ➔ Create queue (Standard).

Encryption: Enable Server-side encryption and select your KMS CMK.

🚨 Access Policy (The First Sandwich Layer): SQS must allow SNS to drop messages into it. Under Access Policy, select Advanced, and ensure this statement exists:

```JSON
{
  "Effect": "Allow",
  "Principal": {
    "Service": "sns.amazonaws.com"
  },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:REGION:ACCOUNT:YourQueueName",
  "Condition": {
    "ArnEquals": {
      "aws:SourceArn": "arn:aws:sns:REGION:ACCOUNT:YourSNSTopicName" 
    }
  }
}
```
## 📣 Step 4: SNS Topic (The Megaphone)
Now build the router.

Go to SNS ➔ Create topic (Standard).

Encryption: Enable it and select your KMS CMK.

🚨 Access Policy (The Second Sandwich Layer): SNS must allow S3 to publish to it. Edit the Access Policy and add:

```JSON
{
  "Effect": "Allow",
  "Principal": {
    "Service": "s3.amazonaws.com"
  },
  "Action": "sns:Publish",
  "Resource": "arn:aws:sns:REGION:ACCOUNT:YourSNSTopicName",
  "Condition": {
    "ArnLike": {
      "aws:SourceArn": "arn:aws:s3:::your-source-bucket-name"
    }
  }
}
```
Create Subscription: After the topic is created, click Create subscription. Choose SQS as the protocol, select your queue, and check the box for "Enable raw message delivery" (this strips the annoying SNS JSON wrapper so Lambda gets the pure S3 event).

## ⚙️ Step 5: Lambda (The Engine)
Now wire the compute that connects the queue to the database.

The IAM Role: Your Lambda Execution Role needs:

AWSLambdaSQSQueueExecutionRole (Managed policy to read SQS).

Inline policy for DynamoDB (dynamodb:PutItem).

Inline policy for KMS (kms:Decrypt, kms:GenerateDataKey on your CMK ARN).

Go to Lambda ➔ Create function (Python/Node). Attach the role.

The Trigger: Click Add trigger, select SQS, and pick your queue. (If it fails to attach here, it means your Lambda IAM role is missing the KMS or SQS permissions!).

Add the DynamoDB Table Name as an Environment Variable.

## 🪣 Step 6: S3 Bucket (The Trigger)
Finally, build the front door.

Go to S3 ➔ Create bucket.

Default Encryption: Enable SSE-KMS, choose your CMK, and Enable Bucket Keys (for FinOps points).

The Event Link: Go to the bucket's Properties tab. Scroll down to Event Notifications ➔ Create event notification.

Name it, select All object create events, scroll to the bottom, select SNS Topic as the destination, and choose your SNS Topic.
(Note: If you get a red "Unable to validate the following destination configurations" error when you click save, it means your SNS Access Policy or KMS Key Policy in Steps 1 & 4 is wrong!)

## 🕵️‍♂️ The Day-Of Troubleshooting Checklist
If you build all of this, upload an image to S3, and nothing appears in DynamoDB, check the pipeline exactly in this order:

S3 to SNS: Did S3 accept the event notification config? If yes, this link is fine.

SNS to SQS: Look at the SQS "Messages Available" metric. Is it sitting at 1? If yes, the message made it to the queue, but Lambda isn't picking it up. If it's 0, your SNS-to-SQS Access policy or KMS policy is blocking it.

SQS to Lambda: Look at CloudWatch Logs for your Lambda. Did it throw an AccessDeniedException for KMS? If so, Lambda can't decrypt the queue.

Lambda to DynamoDB: Did Lambda throw a ProvisionedThroughputExceeded or AccessDenied for DynamoDB? Check the Lambda IAM Execution role again.

## How 2 CMKs Changes the Architecture (The Lambda Pivot)
If you decide (or are forced by the prompt) to use 2 CMKs, the architecture shifts heavily onto your Lambda function's Execution Role.

Lambda becomes the "Cryptographic Bridge" between the two zones. Its IAM policy must explicitly look like this:

kms:Decrypt for Key A (So it can read the incoming message from SQS).

kms:Encrypt and kms:GenerateDataKey for Key B (So it can securely write the processed data into DynamoDB).

If Lambda only has permission for Key A, it will read the queue perfectly, but crash with an AccessDeniedException the millisecond it tries to talk to DynamoDB.


# CloudWatch

## 🪵 1. CloudWatch Logs (The Source of Truth)
This is where application errors (Lambda, API Gateway) and network rejections (VPC Flow Logs) live.

Step 1: Set Retention (Cost Optimization): By default, logs stay forever.

How: Go to Log Groups -> Select your group -> Actions -> Edit retention setting -> Change to 14 or 30 days. (Automatic points!).

Step 2: Create a Metric Filter (Operational Excellence): You did this yesterday! It extracts data from raw text.

How: Log Groups -> Select group -> Metric filters -> Create metric filter.

Filter Pattern: Use ?ERROR ?Exception ?Fail to catch multiple types of errors.

Metric Value: Set to 1. Now, every time "ERROR" appears, CloudWatch plots a "1" on a graph.

Step 3: Logs Insights (The Detective's Cheat Code): When a judge says, "Find the IP address that caused the 500 error," do not scroll through text manually!

How: Go to Logs Insights -> Select your Log Group.

The Query: Use this exact syntax to find the most recent errors:

Plaintext
fields @timestamp, @message
| filter @message like /Error/
| sort @timestamp desc
| limit 20
Click Run Query.

## 📈 2. CloudWatch Metrics (The Heartbeat)
Metrics are the numbers (CPU, Latency, Invocations).

Step 1: Standard vs. Custom: AWS provides standard metrics for free (every 1 minute to 5 minutes). If your script pushes its own data, it's a Custom Metric.

Step 2: High-Resolution Metrics:

How: When pushing a custom metric via the CLI or Boto3, set StorageResolution=1. This tracks data every 1 second instead of 1 minute. (Massive flex for real-time dashboards).

Step 3: Metric Math: (You conquered this!)

How: Go to All Metrics -> Graph metrics. Add two metrics (e.g., m1 = 5xx Errors, m2 = Total Requests).

Add Math: Click "Add math" -> m1 / m2 * 100. This instantly creates a live "Error Rate Percentage" graph.

## 🚨 3. CloudWatch Alarms (The Automation Trigger)
An alarm watches a metric. If the metric crosses a line, the alarm fires.

Step 1: Static vs. Anomaly:

Static: "Fire if CPU > 80%."

Anomaly Detection (The Architect Move): Select "Anomaly detection" instead of a static number. CloudWatch uses Machine Learning to learn your normal traffic pattern and only alarms if traffic acts weirdly. (Judges love seeing ML applied to Ops).

Step 2: The "Action" Trap (Crucial):

An alarm that just turns red is useless. You MUST configure the "Actions" tab.

How: Actions -> Send a message to an SNS Topic (which emails the admins) OR trigger an Auto Scaling Group to add more EC2 instances.

Step 3: Composite Alarms:

How: Create Alarm -> Composite Alarm.

Why: "Only wake me up IF (CPU > 90%) AND (Database Latency > 2 seconds)." This prevents alert fatigue.

## 🖥️ 4. CloudWatch Dashboards (The Command Center)
This is what the judges will look at to see if you succeeded.

Step 1: Cross-Region / Cross-Account Dashboards:

You don't need a separate dashboard for us-east-1 and us-west-2. You can add widgets from multiple regions onto one master screen.

Step 2: Dynamic Widgets:

Don't just add line graphs. Use the Number widget for current error counts, and the Gauge widget for CPU utilization.

Pro-Tip: Add a Text widget (using Markdown) at the top of your dashboard explaining what the dashboard does. "Welcome to the Serverless Admin Dashboard." It shows incredible polish.

## 🕵️‍♂️ 5. CloudWatch Synthetics (The User Experience)
If the prompt asks: "Ensure the website is actually returning a 200 OK status every minute."

Step 1: Create a Canary:

How: Application Monitoring -> Synthetics Canaries -> Create canary.

Blueprint: Use the "Heartbeat monitoring" blueprint.

Target: Paste your API Gateway or ALB URL.

What it does: It spins up a hidden, headless Lambda function that acts like a real Google Chrome browser, hits your website every minute, takes a screenshot, and reports if it failed.


# Secrets Manager and SSM Parameter Store
Blueprint 1: AWS Secrets Manager (The Premium Vault)
Use this if the prompt asks for Auto-Rotation, RDS Integration, or Cross-Account Access.

🏗️ How to Create It (Console)
Go to Secrets Manager ➔ Store a new secret.

Secret type: If it's for RDS, select Credentials for Amazon RDS (this allows auto-rotation). If it's just a random API key, select Other type of secret.

Key/Value pairs: Enter your keys (e.g., Key: password, Value: SuperSecret123!).

Encryption Key: Select your Customer Managed Key (CMK) or leave as the default aws/secretsmanager. Click Next.

Secret name: Give it a path-like name (e.g., prod/database/mysql). Click Next.

Rotation: Leave disabled unless the prompt specifically asks you to configure a rotation Lambda function. Click Store.

🛠️ How to Fetch It (The CLI Cheat Code)
When you SSH into your EC2 instance or write your Bash script, use this exact command to pull the pure JSON string out:

```Bash
aws secretsmanager get-secret-value \
    --secret-id prod/database/mysql \
    --query SecretString \
    --output text
```
🛡️ Blueprint 2: SSM Parameter Store (The FinOps Hero)
Use this if the prompt asks for Cost Optimization (Free), or storing Configuration Data alongside secrets.

🏗️ How to Create It (Console)
Go to Systems Manager ➔ Left Menu: Parameter Store ➔ Create parameter.

Name: It must start with a slash if you want to use hierarchies (e.g., /prod/database/password).

Tier: Select Standard (Free tier!).

Type (The Golden Trap): Select SecureString. (If you select String, it saves the password in plain text and you instantly fail the Security pillar!)

KMS Key ID: Select your CMK or the default alias/aws/ssm.

Value: Type in the actual password. Click Create.

🛠️ How to Fetch It (The CLI Cheat Code)
When you are on your EC2 instance, use this command. Notice the crucial --with-decryption flag. If you forget this, AWS will hand you the encrypted gibberish instead of the password!

```Bash
aws ssm get-parameter \
    --name "/prod/database/password" \
    --with-decryption \
    --query "Parameter.Value" \
    --output text
```
🚨 The Universal IAM "Fault Finding" Trap for Both
If your EC2 instance or Lambda function times out or gets an AccessDeniedException when running those commands, check these two exact permissions in their IAM Execution Role:

If they used Secrets Manager:

secretsmanager:GetSecretValue (Pointed at the Secret ARN)

kms:Decrypt (Pointed at the KMS Key used to encrypt the secret)

If they used SSM Parameter Store:

ssm:GetParameter (Pointed at the Parameter ARN)

kms:Decrypt (Pointed at the KMS Key used to encrypt the parameter)
