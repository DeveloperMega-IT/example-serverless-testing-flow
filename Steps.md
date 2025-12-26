Based on your project plan and conversation history, here are the comprehensive, step-by-step instructions to complete Phase 1: from creating your Lambda function to receiving a confirmation email via SES.

### **Step 1: Create the Lambda Function (The "Logic")**
1.  Open the **AWS Lambda Console** and ensure your region is set to **Mumbai (ap-south-1)**.
2.  Click **Create function**, select **Author from scratch**, and name it `example_function`.
3.  Choose **Python 3.12** (or higher) as the runtime.
4.  In the code editor, you will eventually deploy the "Pro" version of your code to handle data and emails.

### **Step 2: Set up API Gateway (The "Bridge")**
1.  Go to the **API Gateway Console** and create an **HTTP API** (which uses the `$default` stage).
2.  Create a **Route** named `/example` and attach the **GET method** to it.
3.  Set the **Integration Target** for this route to your `example_function`.
4.  **Deploy** the API to a stage to receive your **Invoke URL** (your specific ID is `lv02y8ixo4`).

### **Step 3: Configure IAM Roles (The "Permissions")**
1.  Navigate to the **IAM Console** and find the **Role** automatically created for your `example_function`.
2.  Click **Add permissions** and select **Attach policies**.
3.  Search for and attach these two managed policies:
    *   **AmazonDynamoDBFullAccess**: To allow the function to save user data.
    *   **AmazonSESFullAccess**: To allow the function to send emails.

### **Step 4: Create the DynamoDB Table (The "Storage")**
1.  Open the **DynamoDB Console** in the Mumbai region.
2.  Click **Create table** and use the following settings:
    *   **Table name**: `UserSubmissions` (Note: this is case-sensitive).
    *   **Partition key**: `email` (Type: String).
3.  Keep all other settings as default and click **Create**.

### **Step 5: Verify SES Identity (The "Email Security")**
1.  Go to the **Amazon SES Console** in the Mumbai region.
2.  Under **Verified Identities**, click **Create identity**.
3.  Select **Email address** and enter your verified email: `yourmail@gmail.com`.
4.  **Crucial Step**: Check your Gmail inbox and click the **verification link** sent by AWS; the status must show as "Verified" in green.

### **Step 6: Deploy the "Lambda Pro" Code**
Replace your Lambda function code with this finalized version that connects all services:

```python
import json
import boto3
from botocore.exceptions import ClientError

# Initialize Clients for Mumbai Region
dynamodb = boto3.resource('dynamodb', region_name='ap-south-1')
ses = boto3.client('ses', region_name='ap-south-1')
table = dynamodb.Table('UserSubmissions')

def lambda_handler(event, context):
    # 1. Extract parameters from the GET request URL
    params = event.get('queryStringParameters', {})
    user_name = params.get('name', 'Tony')
    user_email = params.get('email', 'chandruvenkadasalam@gmail.com')

    # MUST be your verified SES email
    MY_VERIFIED_EMAIL = "chandruvenkadasalam@gmail.com"

    try:
        # 2. Save to DynamoDB
        table.put_item(Item={'email': user_email, 'name': user_name})

        # 3. Send Email via SES
        ses.send_email(
            Source=MY_VERIFIED_EMAIL,
            Destination={'ToAddresses': [user_email]},
            Message={
                'Subject': {'Data': 'Phase 1 Success!'},
                'Body': {'Text': {'Data': f"Hi {user_name}, your data is saved in Mumbai!"}}
            }
        )

        return {
            'statusCode': 200,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps("Success! Data saved and Email sent.")
        }
    except ClientError as e:
        return {'statusCode': 500, 'body': json.dumps(f"AWS Error: {e.response['Error']['Message']}")}
```
*Make sure to click **Deploy** after pasting this code.*

### **Step 7: Final Test of the Flow**
To test the complete integration, paste the following URL into your browser, replacing the name if desired:

`https://lv02y8ixo4.execute-api.ap-south-1.amazonaws.com/example?name=Tony&email=chandruvenkadasalam@gmail.com`

**Verification Checklist**:
*   **Browser**: You should see a "Success" message.
*   **DynamoDB**: Check the "Explore items" tab in your `UserSubmissions` table to see the new entry.
*   **Email**: Check your `yourmail@gmail.com` inbox for the "Phase 1 Success!" message.

***

**Analogy for Understanding:**
Think of **IAM Roles** as a **master keycard**. Even though you have built a **Lambda house** with a **DynamoDB room** and an **SES room**, the Lambda "inhabitant" cannot enter those rooms to store things or send mail until you specifically program their keycard (the IAM Policy) to unlock those doors.
