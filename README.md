# Serverless Image Processing Pipeline

This project uses AWS services to resize uploaded images automatically.
Original images are stored in one S3 bucket. A Python Lambda function
uses Pillow to create thumbnails that fit within 200 × 200 pixels.
The thumbnails are saved as JPEG files in a second S3 bucket.

The project also provides an API Gateway endpoint that starts a
Step Functions workflow to process an existing image.

## AWS Services Used

- Amazon S3 stores the original images and resized thumbnails.
- AWS Lambda runs the Python image resizing code.
- AWS Step Functions controls the processing workflow and checks
  whether resizing succeeded.
- Amazon API Gateway provides a POST endpoint to start the workflow.
- Amazon CloudWatch stores Lambda execution logs.
- AWS IAM roles give the services permission to access the required resources.

## Architecture

The application supports two ways to process an image:

1. Automatic processing: uploading an image to the original S3 bucket
   triggers the ResizeImage Lambda function. The function saves a
   thumbnail in the resized images bucket.

2. API processing: a POST request to API Gateway starts the
   ImageProcessingWorkflow state machine. The workflow invokes
   ResizeImage, checks the result, and follows the success or failure branch.

Calling the API for an image that was already processed recreates
the thumbnail at the same output location.

## Project Configuration

- AWS Region: ca-central-1
- Original images bucket: joshjumbo-original-images
- Resized images bucket: joshjumbo-resized-images
- Lambda function: ResizeImage
- Lambda runtime: Python 3.12
- Lambda architecture: x86_64
- Lambda layer: PillowLayer
- State machine: ImageProcessingWorkflow
- State machine ARN:
  arn:aws:states:ca-central-1:515105386780:stateMachine:ImageProcessingWorkflow
- API method: POST
- API endpoint:
  https://9jj6l8ow3a.execute-api.ca-central-1.amazonaws.com/dev/process-image

  ## Prerequisites

- An AWS account with permission to configure S3, Lambda, IAM,
  Step Functions, API Gateway, and CloudWatch.
- Python 3.12 and pip.
- PowerShell for the commands shown below.
- A JPG or PNG image for testing.
- The project source code from this repository.

## Deployment Instructions

### 1. Create the S3 buckets

Create two uniquely named buckets in ca-central-1:
one for original images and one for thumbnails.
Keep Block all public access enabled.

### 2. Configure Lambda permissions

Create a Lambda execution role with AWSLambdaBasicExecutionRole.
Add permission to read objects from the original bucket using
s3:GetObject and write objects to the output bucket using s3:PutObject.

### 3. Prepare the Pillow layer

Use Python 3.12 to install a Linux-compatible Pillow package.
Run these commands from the project root:
```powershell
mkdir pillow-layer
cd pillow-layer

python -m pip install Pillow --platform manylinux2014_x86_64 --implementation cp --python-version 3.12 --only-binary=:all: --target python

Compress-Archive -Path .\python -DestinationPath .\pillow-layer.zip

cd ..
```
Upload pillow-layer.zip as a Lambda layer compatible with
Python 3.12 and x86_64.

### 4. Create the Lambda function

Create ResizeImage using Python 3.12 and x86_64.
Attach the execution role and Pillow layer.
Copy the code from lambda/lambda_function.py into the function
and select Deploy.

Set the memory to 512 MB and the timeout to 30 seconds.

Configure these environment variables:

- ORIGINAL_BUCKET: the original images bucket name.
- OUTPUT_BUCKET: the resized images bucket name.

### 5. Configure the automatic upload trigger

Add an S3 trigger to ResizeImage for all object create events
in the original images bucket.

### 6. Create the Step Functions workflow

Create a Standard workflow using the definition in
statemachine/image-processing.asl.json.

Replace the Lambda function ARN in the definition with your own.
Use an execution role that allows lambda:InvokeFunction
on ResizeImage.

The workflow includes a Task state, a Choice state,
a Succeed state, and a Fail state.

### 7. Configure API Gateway

Create a Regional REST API with the resource /process-image
and a POST method.

Configure an AWS service integration for Step Functions,
using POST and the StartExecution action.

Use an API Gateway execution role that allows
states:StartExecution on the state machine.

Add the application/json mapping template from
statemachine/api-request-mapping.vtl.
Update its stateMachineArn to your own state machine ARN.

Deploy the API to a stage named dev.

## Testing Automatic Processing

1. Open the original images bucket in S3.
2. Upload a JPG or PNG image larger than 200 × 200 pixels.
3. Open the resized images bucket and refresh the object list.
4. Open the thumbnails folder.
5. Download the generated JPEG and check its dimensions.

The thumbnail keeps the image's proportions, and neither dimension
exceeds 200 pixels.

For example, Schedule.png produces thumbnails/Schedule.jpg.

## Testing the API

First, upload Schedule.png to joshjumbo-original-images.

Run the following command in PowerShell:

```powershell
$body = @{
    bucket = "joshjumbo-original-images"
    key = "Schedule.png"
} | ConvertTo-Json

Invoke-RestMethod `
    -Method Post `
    -Uri "https://9jj6l8ow3a.execute-api.ca-central-1.amazonaws.com/dev/process-image" `
    -ContentType "application/json" `
    -Body $body
```

The response should contain executionArn and startDate.
This confirms that the workflow started.

Open Step Functions and check that the execution finishes
with status Succeeded. Then check the output bucket for
thumbnails/Schedule.jpg.

## Testing the API

First, upload Schedule.png to joshjumbo-original-images.

Run the following command in PowerShell:

```powershell
$body = @{
    bucket = "joshjumbo-original-images"
    key = "Schedule.png"
} | ConvertTo-Json

Invoke-RestMethod `
    -Method Post `
    -Uri "https://9jj6l8ow3a.execute-api.ca-central-1.amazonaws.com/dev/process-image" `
    -ContentType "application/json" `
    -Body $body
```

The response should contain executionArn and startDate.
This confirms that the workflow started.

Open Step Functions and check that the execution finishes
with status Succeeded. Then check the output bucket for
thumbnails/Schedule.jpg.

## Troubleshooting

### Python launcher was not recognized

PowerShell could not find the py command.
I used python instead of py -3.12.

### The source bucket was not allowed

The bucket name in the test input did not match the
ORIGINAL_BUCKET environment variable.
Both values must contain the same original bucket name.

### InvalidArn when starting the workflow

The API mapping template contained an IAM role ARN.
The stateMachineArn field must contain the Step Functions
state machine ARN.

### AccessDeniedException for StartExecution

The APIGatewayToImageWorkflow role needed an inline policy
allowing states:StartExecution on ImageProcessingWorkflow.

### Missing Authentication Token

The request URL ended at /dev.
The complete URL must include /dev/process-image.