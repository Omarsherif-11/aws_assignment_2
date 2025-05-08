# Orders Processing System

## Architecture

The system consists of:

- **Amazon SNS Topic (`OrdersTopic`)**: Publishes order messages to an SQS queue with raw message delivery enabled.
- **Amazon SQS Queue (`OrderQueue`)**: Queues messages with a 300-second visibility timeout and 4-day retention. Failed messages (after 3 attempts) are sent to `DeadLetterQueue`.
- **AWS Lambda Function (`processOrder`)**: Processes messages from `OrderQueue`, validates order data, and writes to DynamoDB. Uses Node.js 18.x with a 3-second timeout.
- **Amazon DynamoDB Table (`Orders`)**: Stores order data with `orderId` as the partition key, using on-demand billing and AWS-managed encryption.
- **Dead Letter Queue (`DeadLetterQueue`)**: Stores failed messages for debugging.

![Architecture Diagram](https://github.com/omarsherif-11/aws_assignment_2/blob/main/images/architecture-diagram.png)

## Prerequisites

- An AWS account with permissions for SNS, SQS, Lambda, DynamoDB, CloudFormation, and IAM.

## Setup

Follow these steps to deploy the system using the AWS Management Console. All steps are performed in the browser, no command-line tools required.

### Step 1: Prepare the CloudFormation Template

1. **Download the Template**:
   - Locate the `orders-system-template.yaml` file in this repository.
   - Click the file, then click **Raw** to view the plain text.
   - Right-click and select **Save As** to download it as `orders-system-template.yaml`.
2. **Verify the Template**:
   - Open the file in a text editor (e.g., Notepad) to ensure it contains YAML content with resources like `SNSTopic`, `SQSQueue`, `OrderProcessorLambda`, and `OrdersTable`.
   - Do not modify the file unless instructed.

### Step 2: Deploy the CloudFormation Stack

1. **Log in to AWS Management Console**:

   - Open your browser and go to [https://console.aws.amazon.com](https://console.aws.amazon.com).
   - Sign in with your AWS account credentials.
   - Select the `us-east-1` region (top-right corner) to match the system’s configuration.

2. **Navigate to CloudFormation**:

   - In the AWS Console, click \*\*Services - Search for “CloudFormation” and select it.
   - Click **Stacks** in the left menu.

3. **Create a New Stack**:

   - Click **Create stack** > **With new resources (standard)**.

4. **Upload the Template**:

   - Select **Upload a template file**.
   - Click **Choose file** and select `orders-system-template.yaml` from your computer.
   - Click **Next**.

5. **Specify Stack Details**:

   - **Stack name**: Enter a unique name (e.g., `OrdersSystem`).
     - Note: If `OrdersTopic`, `OrderQueue`, `processOrder`, or `Orders` already exist in `us-east-1`, use a different name (e.g., `OrdersSystemTest`) to avoid conflicts.
   - **Parameters**:
     - **Environment**: Choose `dev` (or `staging`, `prod` if preferred). This sets the environment for resource naming.
   - Click **Next**.

6. **Configure Stack Options**:

   - Leave defaults for **Tags**, **Permissions**, and **Stack creation options** unless you have specific requirements (e.g., add a tag like `Project=Orders`).
   - Click **Next**.

7. **Review and Create**:

   - Review the stack details and parameters.
   - Check the box to acknowledge IAM resource creation (the template creates a role: `lambdaRole`).
   - Click **Create stack**.

8. **Monitor Stack Creation**:
   - Wait 2-5 minutes for the stack to deploy.
   - Check the **Events** tab for progress. If the status reaches `CREATE_COMPLETE`, the deployment succeeded.
   - If errors occur (e.g., resource name conflicts), note the error message, delete the stack (via **Actions** > **Delete stack**), and contact the project maintainer.

### Step 3: Verify Deployed Resources

1. **SNS Topic**:

   - Go to **SNS** > **Topics**.
   - Find `OrdersTopic` and click it.
   - Verify the subscription to `OrderQueue` (ARN ending in `OrderQueue`) with raw message delivery enabled.

2. **SQS Queue**:

   - Go to **SQS** > **Queues**.
   - Find `OrderQueue` and click it.
   - Check:
     - **Details**: Visibility timeout (300 seconds), retention period (4 days).
     - **Dead-letter queue**: Linked to `DeadLetterQueue` with max receives (3).
     - **Lambda triggers**: Linked to `processOrder`.
     - **Encryption**: Amazon SQS-managed key.
   - Verify `DeadLetterQueue` exists similarly.

3. **Lambda Function**:

   - Go to **Lambda** > **Functions**.
   - Find `processOrder` and click it.
   - Check:
     - **Configuration** > **General configuration**: Runtime (Node.js 18.x), timeout (3 seconds), memory (128 MB).
     - **Configuration** > **Triggers**: Linked to `OrderQueue` with batch size 10.
     - **Configuration** > **Permissions**: Role named `lambdaRole`.

4. **DynamoDB Table**:
   - Go to **DynamoDB** > **Tables**.
   - Find `Orders` and click it.
   - Check:
     - **Overview**: Partition key (`orderId`, String), no sort key, on-demand billing.
     - **Additional settings**: AWS-managed encryption.

### Step 4: Test the System

1. **Publish a Test Message**:

   - Go to **SNS** > **Topics** > `OrdersTopic`.
   - Click **Publish message**.
   - Enter a JSON message:
     ```json
     {
       "orderId": "test123",
       "userId": "user1",
       "itemName": "Laptop",
       "quantity": 1,
       "status": "pending",
       "timestamp": "2025-05-03T12:00:00Z"
     }
     ```
   - Click **Publish message**.

2. **Verify Queue Processing**:

   - Go to **SQS** > **Queues** > `OrderQueue`.
   - Click **Send and receive messages** > **Poll for messages**.
   - If the message was processed, it may not appear (due to Lambda’s instant processing).
   - Check `DeadLetterQueue` for failed messages (e.g., if required fields were missing).

3. **Check DynamoDB**:

   - Go to **DynamoDB** > **Tables** > **Orders** > **Explore table items**.
   - Look for an item with `orderId: test123` and the expected fields (`userId`, `itemName`, `quantity`, `status`. `timestamp`).

4. **Monitor Lambda Logs**:
   - Go to **CloudWatch** > **Log groups** > `/aws/lambda/processOrder`.
   - Click the latest log stream.
   - Look for logs like:
     - `Received event: ...` (shows the SQS event).
     - `Processing order: test123`.
     - `Order saved to DynamoDB: test123` (success) or `Error processing message: ...` (failure).

## Usage

- **Publish Orders**:
  - Send JSON messages to `OrdersTopic` with fields: `orderId`, `userId`, `itemName`, `quantity`, `status`, `timestamp`.
  - Example:
    ```json
    {
      "orderId": "123",
      "userId": "user1",
      "itemName": "Laptop",
      "quantity": 1,
      "status": "pending",
      "timestamp": "2025-05-03T12:00:00Z"
    }
    ```
- **Monitor**:
  - Check `OrderQueue` and `DeadLetterQueue` in the SQS Console.
  - View `processOrder` logs in CloudWatch.
  - Query the `Orders` table in DynamoDB.
