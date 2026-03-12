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