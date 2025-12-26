To transition your project so that the **website runs locally** on your computer instead of using S3, follow these consolidated steps based on your flow and conversation history. By running the `index.html` file directly from your desktop, you can still trigger the full backend flow: **Local Form → API Gateway → Lambda → DynamoDB → SES**.

### **Step 1: Create the Lambda Function (The Logic)**
1.  Open the **AWS Lambda Console** in the **Mumbai (ap-south-1)** region.
2.  Click **Create function**, select **Author from scratch**, and name it `example_function`.
3.  Choose **Python 3.12** (or higher) as the runtime.
4.  In the code editor, you will eventually paste the "Lambda Pro" code provided in Step 6.

### **Step 2: Set up API Gateway (The Bridge)**
1.  In the **API Gateway Console**, create an **HTTP API** (which defaults to the `$default` stage).
2.  Create a **Route** named `/example` and attach a **GET method** to it.
3.  Set the **Integration Target** for this route to your `example_function`.
4.  Note your **Invoke URL**. Based on your records, your specific ID is `lv02y8ixo4`, making your base URL: `https://lv02y8ixo4.execute-api.ap-south-1.amazonaws.com/example`.

### **Step 3: Configure IAM Roles (Permissions)**
1.  Navigate to the **IAM Console** and find the **Role** attached to your `example_function`.
2.  Click **Add permissions** and select **Attach policies**.
3.  Search for and attach these two policies to "unlock" your Lambda:
    *   **AmazonDynamoDBFullAccess**: To allow the function to save data.
    *   **AmazonSESFullAccess**: To allow the function to send the confirmation email.

### **Step 4: Create the DynamoDB Table (Storage)**
1.  Open the **DynamoDB Console** in Mumbai.
2.  Click **Create table** and use these exact settings (they are case-sensitive):
    *   **Table name**: `UserSubmissions`.
    *   **Partition key**: `email` (Type: String).
3.  Wait for the table status to turn **Active**.

### **Step 5: Verify SES Identity (Email Security)**
1.  Go to the **Amazon SES Console** in Mumbai.
2.  Under **Verified Identities**, click **Create identity**.
3.  Select **Email address** and enter: `chandruvenkadasalam@gmail.com`.
4.  **Important**: Check your Gmail inbox and click the **verification link** from AWS; it must show a green "Verified" status to work.

### **Step 6: Deploy the "Lambda Pro" Code**
Replace all code in your Lambda function with this version, which includes **CORS headers** necessary for your local website to communicate with AWS:

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
    user_email = params.get('email', 'yourmail@gmail.com')

    # MUST be your verified SES email
    MY_VERIFIED_EMAIL = "yourmail@gmail.com"

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

        # 4. Return success with CORS headers for local testing
        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Methods': 'GET,OPTIONS'
            },
            'body': json.dumps("Success! Data saved and Email sent.")
        }
    except ClientError as e:
        return {'statusCode': 500, 'body': json.dumps(f"AWS Error: {e.response['Error']['Message']}")}
```
*Click **Deploy** after updating the code.*

### **Step 7: Create and Run the Local Website**
Since you are **not using S3**, you will run the website directly from your computer.
1.  Create a file on your desktop named `index.html`.
2.  Paste the following HTML/JavaScript code. This uses the **Fetch API** to call your API Gateway:

```html
<!DOCTYPE html>
<html>
<body>
    <h2>AWS Local Form Test</h2>
    <form id="userForm">
        Name: <input type="text" id="name"><br><br>
        Email: <input type="text" id="email" value="chandruvenkadasalam@gmail.com"><br><br>
        <button type="button" onclick="submitData()">Submit to AWS</button>
    </form>

    <script>
        function submitData() {
            const name = document.getElementById('name').value;
            const email = document.getElementById('email').value;
            // Your specific Invoke URL
            const url = `https://lv02y8ixo4.execute-api.ap-south-1.amazonaws.com/example?name=${name}&email=${email}`;

            fetch(url)
            .then(response => response.json())
            .then(data => alert(data))
            .catch(error => console.error('Error:', error));
        }
    </script>
</body>
</html>
```
3.  **To run it**: Simply **double-click** `index.html` to open it in your browser.
4.  Enter a name and your **verified email**, then click submit. 

### **Step 8: Final Verification**
*   **Local Browser**: You should receive a "Success" alert.
*   **DynamoDB**: Go to the "Explore items" tab in your `UserSubmissions` table to see the new data.
*   **Gmail**: Check your inbox for the "Phase 1 Success!" confirmation.

***

**Analogy for Understanding:**
Think of your **local `index.html`** as a **remote control**. Even though the remote control is in your hand at home (local), it sends an invisible signal (the API call) to the **TV (AWS Lambda)** in the clouds. As long as the TV is plugged in and has permission to talk to the **DVD player (DynamoDB)** and the **Antenna (SES)**, the whole system works without the remote ever needing to be "hosted" inside the TV itself.
