# AWS Lambda Reference Architecture: Mobile Backend

## Introduction

The Mobile Backend reference architecture ([diagram](https://s3.amazonaws.com/awslambda-reference-architectures/mobile-backend/lambda-refarch-mobilebackend.pdf)) demonstrates how to use AWS Lambda along with other services to build a serverless backend for a mobile application. The specific example application provided in this repository enables users to upload photos and notes using Amazon Simple Storage Service (Amazon S3) and Amazon API Gateway respectively. The notes are stored in Amazon DynamoDB, and are processed asynchronously using DynamoDB streams and a Lambda function to add them to an Amazon CloudSearch domain. In addition to the source code for the Lambda functions, this repository also contains a prototype iOS application that provides examples for how to use the AWS Mobile SDK for iOS to interface with the backend resources defined in the architecture.

## Running the example

To run the full example application, you must first deploy the backend resources, and then compile and run the example iOS application.

### Deploying the Backend

The provided AWS CloudFormation template creates most of the backend resources that you need for this example, but you still need to create the Amazon CloudSearch domain, API Gateway REST API, and Cognito identity pool outside of AWS CloudFormation.

#### Step 1: Create a CloudSearch Domain

1. Using the [AWS CLI](https://aws.amazon.com/cli/), create a new CloudSearch domain providing a domain name of your choice.

    ```
    aws cloudsearch create-domain --domain-name [YOUR_DOMAIN_NAME]
    ```

1. Note the ARN of the new domain in the output document. You will use this as an input when launching the CloudFormation stack.

1. Define indexes for the `headline` and `text` fields.

    ```
    aws cloudsearch define-index-field --name headline --type text --domain-name [YOUR_DOMAIN_NAME]
    aws cloudsearch define-index-field --name text --type text --domain-name [YOUR_DOMAIN_NAME]
    ```

#### Step 2: Create an API Gateway REST API

1. Using the [AWS CLI](https://aws.amazon.com/cli/), create a new API providing a name of your choice.

    ```
    aws apigateway create-rest-api --name [YOUR_API_NAME]
    ```

1. Note the `API ID` provided in the output document. You will use this as an input when launching the CloudFormation stack.

#### Step 3: Create an Amazon Cognito Identity Pool

1. Using the [AWS CLI](https://aws.amazon.com/cli/), create a new identity pool providing a name of your choice.

    ```
    aws cognito-identity create-identity-pool --allow-unauthenticated-identities --identity-pool-name [YOUR_POOL_NAME]
    ```

1. Note the `IdentityPoolId` from the output document. You will use this as a parameter when launching the CloudFormation stack.

#### Step 4: Launch the CloudFormation Template

You can deploy the entire example in the us-east-1 region using the provided CloudFormation template and S3 bucket. If you would like to deploy the template to a different region, you must create an Amazon S3 bucket in that region, and then copy the template and Lambda function definitions into it.

Choose **Launch Stack** to launch the template in the us-east-1 region in your account:

[![Launch Lambda Mobile Backend into North Virginia with CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=lambda-mobile-backend&templateURL=https://s3.amazonaws.com/awslambda-reference-architectures/mobile-backend/mobile-backend.template)

When prompted, enter the parameter values for the CloudSearch domain, API Gateway REST API, and Amazon Cognito identity pool resources you created in the previous steps.

Details about the resources created by this template are provided in the *CloudFormation Template Resources* section of this document.

#### Step 5: Update Your API Gateway REST API

After you have created the CloudFormation stack, you need to update the API you created previously to use the newly created `NotesApiFunction`.

1. In the [Amazon API Gateway Console](https://console.aws.amazon.com/apigateway/home?region=us-east-1#/apis), choose your API.
1. Choose **Create Resource** to create a new child resource under /.
1. Type `notes` as the resource name and `/notes` as the resource path.
1. Choose **Create Resource**.
1. With the new `/notes` resource selected, choose **Create Method**.
1. Choose `POST` and select the check box.
1. Choose **Lambda Function** as the integration type and then choose the region where you launched the CloudFormation stack as the Lambda region.
1. in **Lambda Function**, type **`NotesApiFunction`** and then select the function created by the CloudFormation Stack.
1. Choose **Save** and grant API Gateway permissions to execute the Lambda function.
1. Choose **Method Request** to edit the request configuration.
1. For **Authorization type**, select `AWS_IAM`.
1. For **API Key Required**, select `true`. 
1. Choose **Deploy API**.
1. For **Deployment stage**, choose `New Stage` and then type a name in **Stage name**.
1. Note the **Invoke URL** for the new stage. You will use this value when running the sample iOS app.

#### Step 6: Create an API Key

1. In the [Amazon API Gateway Console](https://console.aws.amazon.com/apigateway/home?region=us-east-1#/apis), choose **APIs**, and then choose **API Keys**.
1. Choose **Create API Key**.
1. Type a name for the key and then select **Enabled**.
1. Choose **Save**
1. In the **API Stage Association** section, choose your API and then choose the stage that you created in the previous step.
1. Choose **Add**.
1. Note the **API key**. You will use this when running the mobile application.

#### Step 7: Update Your Amazon Cognito Identity Pool

1. In the [Amazon Cognito Console](https://console.aws.amazon.com/cognito/home?region=us-east-1), select your identity pool.
1. Choose **Edit Identity Pool**.
1. For both the **Unauthenticated role** and the **Authenticated role**, select the **MobileClientRole** created by the CloudFormation stack. The full ARN for the role is provided in the outputs of the stack.
1. Choose **Save Changes**.


### Running the Sample iOS Application

#### Prerequisites

To run the provided iOS sample application, you must be running Mac OS X 10.10 (Yosemite) or a more recent version. You must also have the latest version of [Xcode](https://itunes.apple.com/us/app/xcode/id497799835) and [Cocoa Pods](https://cocoapods.org/) installed.

#### Building and Running the Application

1. Check out or download the source code for the **ios-sample** in this repository.
1. Update `MobileBackendIOS/Constants.swift` with the values for your backend deployment. Most of the values can be found in the outputs of the CloudFormation stack. The API Gateway key and endpoint URL values are available in the details for your API in the AWS Management Console.
1. Run Cocoa Pods from the root `ios-sample` directory.

    ```
    pod install
    ```

1. Open the generated `MobileBackendIOS.xcworkspace` file in Xcode.

    ```
    open -a Xcode MobileBackendIOS.xcworkspace
    ```

1. Build and run the project from Xcode by clicking the play button at the top of the window.

## Testing the Application

The sample application provides two features: uploading an image and posting a note.

### To upload an image

1. Choose **Upload Image** in the application.
1. Choose the camera icon, select an image from the camera roll, and then choose **Choose**.
1. Choose the **Upload** button.

#### Validating that the image was uploaded

You should see a log entry that the image has been uploaded to Amazon S3 in the output pane of Xcode.

You can also browse the bucket created by the CloudFormation stack using the AWS Management Console to verify that the image was correctly uploaded.

### To post a note

1. Choose **Post a Note**.
1. In the note, type a headline and text.
1. Choose **Save Note**.

#### Validating that the note was posted

You should see a log entry that the note was saved successfully in the output pane of Xcode.

When the note is uploaded, the `NotesApiFunction` is invoked by the mobile application. You can view the logs for this function in Amazon CloudWatch.

When the function is successfully invoked, it adds an entry to the DynamoDB table created in the CloudFormation stack. You can verify that the note you posted in the application has been persisted in the created table.

Finally, when the note is persisted in the DynamoDB table a record is added to the table's stream which is in turn processed by the `DynamoStreamHandlerFunction`. You can view the logs for this function in CloudWatch and verify that a new document has been added to the CloudSearch domain you created.


## Cleaning Up the Application Resources

To remove all resources created by this example, do the following:

1. Delete all objects from the S3 bucket created by the CloudFormation stack.
1. Delete the CloudFormation stack.
1. Delete the Amazon Cognito identity pool, API Gateway, and CloudSearch domain.
1. Delete the CloudWatch log groups associated with each Lambda function created by the CloudFormation stack.

## CloudFormation Template Resources

### Lambda Functions

- **NotesApiFunction** - A function to handle posted notes from the mobile application via API Gateway.

- **SearchApiFunction** - A function that uses the CloudSearch domain to find indexed notes based on search terms.

- **DynamoStreamHandlerFunction** - A function that adds an indexed document to the provided CloudSearch domain based on records in the `PhotoNotesTable` stream.

### AWS Identity and Access Management (IAM) Roles

- **NotesApiRole** - A role for the `NotesApiFunction`. This role grants permission for logging and working with items in the `PhotoNotesTable`.

- **SearchApiRole** - A role for the `SearchApiFunction`. This role grants permissions for logging and searching the provided CloudSearch domain.

- **DynamoStreamHandlerRole** - A role for the DynamoStreamHandlerFunction. This role grants permissions for logging and adding documents to the provided CloudSearch domain.

- **MobileClientRole** - A role used by your Amazon Cognito identity pool for both unauthenticated and authenticated users. This role provides access to the provided API Gateway REST API as well as permissions for putting objects to the `MobileUploadsBucket`.

### Other Resources

- **MobileUploadsBucket** - An S3 bucket for user uploaded photos.

- **CloudFrontDistribution** - A CDN distribution with the `MobileUploadsBucket` configured as an origin.

- **PhotoNotesTable** - A DynamoDB table that stores notes uploaded by users from the mobile application.

## License

This reference architecture sample is licensed under Apache 2.0.
