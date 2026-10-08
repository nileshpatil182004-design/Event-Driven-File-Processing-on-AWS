
# Event-Driven File Processing on AWS

### Amazon S3 · AWS Lambda · Amazon CloudWatch

**A serverless project that automatically detects file uploads and
records object details in CloudWatch Logs.**

![AWS](https://img.shields.io/badge/Cloud-AWS-232F3E?logo=amazonaws&logoColor=white)
![Amazon
S3](https://img.shields.io/badge/Storage-Amazon%20S3-569A31?logo=amazons3&logoColor=white)
![AWS
Lambda](https://img.shields.io/badge/Compute-AWS%20Lambda-FF9900?logo=awslambda&logoColor=white)
![CloudWatch](https://img.shields.io/badge/Monitoring-CloudWatch-BC4C00?logo=amazoncloudwatch&logoColor=white)
![Python](https://img.shields.io/badge/Language-Python%203-3776AB?logo=python&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-6F42C1)
:::

------------------------------------------------------------------------

## 📌 Project Summary

This project demonstrates an **event-driven, serverless workflow** on
Amazon Web Services (AWS).

When a user uploads a file to an Amazon S3 bucket, an S3 object-creation
event automatically invokes an AWS Lambda function. The function reads
the event payload and writes useful information---such as the bucket
name, object key, event type, AWS Region, and object size---to Amazon
CloudWatch Logs.

The goal is to show how AWS services can work together without running a
continuously active server.

> **Verification note:** The project is complete only after a test
> upload is visible in the S3 Objects list and the corresponding event
> details are confirmed in CloudWatch Logs. Add your own final
> screenshots to document that result.

## 🎯 Objectives

-   Configure an Amazon S3 bucket as an event source.
-   Create a Python-based AWS Lambda function.
-   Trigger Lambda automatically when an object is created in S3.
-   Extract and log bucket and object metadata from the S3 event.
-   Validate the event flow using CloudWatch Logs.
-   Document the implementation and evidence in a GitHub repository.

## 🏛️ Architecture

``` text
                  User uploads a file
                           |
                           v
                  +------------------+
                  |   Amazon S3      |
                  |   Input Bucket   |
                  +------------------+
                           |
                           | ObjectCreated event
                           v
                  +------------------+
                  |   AWS Lambda     |
                  | s3-file-event-   |
                  | logger           |
                  +------------------+
                           |
                           | Execution logs
                           v
                  +------------------+
                  | Amazon CloudWatch|
                  |     Logs         |
                  +------------------+
                           |
                           v
              Bucket name, object key, size,
                event type, Region, request ID
```

### Event flow

1.  A user uploads a file, such as `hello.txt`, to the S3 bucket.
2.  Amazon S3 emits an object-creation event.
3.  The configured S3 notification invokes the Lambda function.
4.  Lambda parses the event record and prints the object details.
5.  CloudWatch Logs stores the Lambda execution output.
6.  The user verifies the result in the relevant CloudWatch log stream.

## 🧰 AWS Services

  -----------------------------------------------------------------------
  Service                             Role in this project
  ----------------------------------- -----------------------------------
  **Amazon S3**                       Stores the test object and acts as
                                      the event source.

  **AWS Lambda**                      Executes Python code in response to
                                      an S3 object-creation event.

  **Amazon CloudWatch Logs**          Captures and displays Lambda
                                      execution logs.

  **AWS IAM**                         Controls the Lambda execution
                                      permissions and allows S3 to invoke
                                      the function.
  -----------------------------------------------------------------------

## ⚙️ Resource Configuration

The following names reflect the setup used during the walkthrough.
Update them if your actual AWS resources use different names.

  Resource / setting     Example value
  ---------------------- ----------------------------------------
  AWS Region             `ap-south-1` --- Asia Pacific (Mumbai)
  S3 bucket              `event-drive-project-2026-unique123`
  Lambda function        `s3-file-event-logger`
  S3 event type          `All object create events`
  CloudWatch log group   `/aws/lambda/s3-file-event-logger`
  Test object            `hello.txt`
  Lambda runtime         Supported Python 3 runtime

**Important:** S3 bucket names are globally unique. The bucket name
above is an example from the walkthrough; replace it with the name of
your own bucket if it differs.

## 🛠️ Implementation Guide

### Step 1 --- Create or select an S3 bucket

1.  Sign in to the [AWS Management
    Console](https://console.aws.amazon.com/).
2.  Open **Amazon S3 → Buckets**.
3.  Create a bucket or select an existing bucket suitable for this
    project.
4.  Keep **Block Public Access** enabled. Public access is not needed
    for this workflow.
5.  Note the bucket name and Region.

### Step 2 --- Create the Lambda function

1.  Open **AWS Lambda → Functions → Create function**.
2.  Choose **Author from scratch**.
3.  Set the function name to `s3-file-event-logger`.
4.  Choose a supported Python runtime.
5.  Configure an execution role that has permission to write logs to
    CloudWatch. The AWS-managed policy `AWSLambdaBasicExecutionRole` is
    commonly used for basic Lambda logging.
6.  Create the function.

### Step 3 --- Add the Lambda code

Open the function's **Code** tab, select `lambda_function.py`, replace
its contents with the code below, and click **Deploy**.

``` python
import json
import urllib.parse
from datetime import datetime, timezone


def lambda_handler(event, context):
    print("===== S3 EVENT RECEIVED =====")
    print("Invocation Time:", datetime.now(timezone.utc).isoformat())
    print("Lambda Request ID:", context.aws_request_id)

    records = event.get("Records", [])

    if not records:
        print("No S3 event records found.")
        print("Event:", json.dumps(event))
        return {
            "statusCode": 400,
            "body": "No S3 event records found"
        }

    for record in records:
        event_name = record.get("eventName", "Unknown")
        region = record.get("awsRegion", "Unknown")

        s3_data = record.get("s3", {})
        bucket_data = s3_data.get("bucket", {})
        object_data = s3_data.get("object", {})

        bucket_name = bucket_data.get("name", "Unknown")
        object_key = object_data.get("key", "Unknown")
        object_size = object_data.get("size", "Unknown")

        # S3 event object keys are URL-encoded.
        if object_key != "Unknown":
            object_key = urllib.parse.unquote_plus(object_key)

        print("Event Name:", event_name)
        print("AWS Region:", region)
        print("Bucket Name:", bucket_name)
        print("Object Name:", object_key)
        print("Object Size:", object_size)

    print("===== S3 EVENT PROCESSING COMPLETED =====")

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "S3 event processed successfully",
            "records_processed": len(records)
        })
    }
```

### Step 4 --- Add the S3 trigger

1.  Open the Lambda function and select **Add trigger**.
2.  Choose **S3** as the source.
3.  Select your project bucket.
4.  Choose **All object create events**.
5.  Leave **Prefix** and **Suffix** empty to receive notifications for
    all object-creation keys.
6.  Review the recursive-invocation acknowledgement if the console
    requires it. This function only writes logs and does not write
    objects back to S3, so this workflow does not create an S3-to-Lambda
    upload loop.
7.  Select **Add** and verify that the S3 trigger appears in the Lambda
    function overview.

**Configuration requirements**

-   The S3 bucket and Lambda function must be in the same AWS Region.
-   S3 must be allowed to invoke the Lambda function. The Lambda console
    can configure the required resource permission when adding the
    trigger.
-   Avoid overlapping S3 notification configurations for the same event
    types and object-key filters.

### Step 5 --- Upload a test file

Create a text file named `hello.txt` with content similar to:

``` text
Hello AWS!

This file tests the S3-to-Lambda event-driven workflow.
```

Then:

1.  Open **Amazon S3 → your bucket → Objects**.
2.  Select **Upload → Add files**.
3.  Select `hello.txt`.
4.  Click **Upload** and wait for the upload to finish.
5.  Confirm that `hello.txt` appears in the bucket's Objects list.

**Private-bucket note:** Opening an object URL directly in a browser may
return `AccessDenied` if the object is private and the request is not
authorized. This does not by itself mean the upload failed. Check the
object in the S3 console and keep the bucket private.

### Step 6 --- Verify CloudWatch Logs

1.  Open **Amazon CloudWatch → Logs → Log groups**.
2.  Open `/aws/lambda/s3-file-event-logger`.
3.  Open the newest log stream.
4.  Inspect the log events and confirm that they correspond to the test
    upload.

Example output (actual values will differ):

``` text
===== S3 EVENT RECEIVED =====
Invocation Time: 2026-10-08T...
Lambda Request ID: ...
Event Name: ObjectCreated:Put
AWS Region: ap-south-1
Bucket Name: event-drive-project-2026-unique123
Object Name: hello.txt
Object Size: ...
===== S3 EVENT PROCESSING COMPLETED =====
```

A new log stream alone does **not** prove that the S3 event was
processed. Confirm that the expected event name, bucket name, and object
key appear in its log events.

## 📸 Screenshots

Create a folder named `screenshots` in the repository and save your own
AWS Console screenshots using these filenames.

  -----------------------------------------------------------------------
  Filename                            Evidence to capture
  ----------------------------------- -----------------------------------
  `01-s3-bucket.png`                  S3 bucket visible in the bucket
                                      list or bucket overview.

  `02-lambda-function.png`            Lambda function overview showing
                                      the function name.

  `03-lambda-code.png`                Deployed Python code in the Lambda
                                      editor.

  `04-s3-trigger.png`                 S3 trigger attached to the Lambda
                                      function.

  `05-file-upload.png`                `hello.txt` visible in the S3
                                      Objects list after upload.

  `06-cloudwatch-logs.png`            Actual CloudWatch log events
                                      showing the bucket and object
                                      details.
  -----------------------------------------------------------------------

### Screenshot gallery

Add the screenshots to the `screenshots/` folder. The images will render
below once the files have been committed to GitHub.

#### 1. S3 bucket

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/s3-bucket.png)

#### 2. Lambda function

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/lambda-function.png)

#### 3. Lambda Python code

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/Lambda-Code.png)

#### 4. S3 trigger

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/s3-Trigger%20-configuration.png)

#### 5. Uploaded test file

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/S3%20Test-file-Upload.png)

#### 6. Browser - hello.txt - Testing

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/Browser-%20hello.txt-Testing.png)

#### 7. CloudWatch event logs

![image alt](https://github.com/nileshpatil182004-design/Event-Driven-File-Processing-on-AWS/blob/c357c6f7b5a9092e3d7f5742199a8014c08113f7/Cloudwatch-logs.png)

## ✅ Validation Checklist

Use this checklist before marking the project complete.

-   [ ] S3 bucket exists in the intended Region.
-   [ ] Lambda function is deployed successfully.
-   [ ] Lambda execution role has CloudWatch logging permissions.
-   [ ] S3 trigger is attached to the correct Lambda function.
-   [ ] `hello.txt` is visible in the S3 Objects list.
-   [ ] CloudWatch logs show the corresponding S3 event.
-   [ ] Screenshots show the actual configuration and successful test
    result.




## 📚 Key Learnings

By completing this project, you practise:

-   Event-driven and serverless architecture concepts.
-   Amazon S3 object-created event notifications.
-   AWS Lambda functions and Python event processing.
-   Reading and decoding S3 object keys from event records.
-   CloudWatch Logs for execution visibility and troubleshooting.
-   IAM permissions and basic AWS security practices.
-   Documenting practical cloud work for a technical portfolio.

## 👤 Author

** Nilesh Pradeep Patil
GitHub: `https://github.com/nileshpatil182004-design`

------------------------------------------------------------------------

**Built for learning AWS serverless and event-driven architecture.**
