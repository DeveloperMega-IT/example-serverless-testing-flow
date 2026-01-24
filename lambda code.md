import json
import boto3
import uuid
from datetime import datetime
from boto3.dynamodb.conditions import Attr

# AWS clients
dynamodb = boto3.resource('dynamodb')
ses = boto3.client('ses')

# Config
TABLE_NAME = "LoanApplications"
SENDER_EMAIL = "companyusage2026@gmail.com"
ADMIN_EMAIL = "companyusage2026@gmail.com"

def lambda_handler(event, context):

    # -------- 1. CORS PREFLIGHT --------
    if event.get("requestContext", {}).get("http", {}).get("method") == "OPTIONS":
        return {
            "statusCode": 200,
            "headers": {
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Headers": "Content-Type",
                "Access-Control-Allow-Methods": "POST,OPTIONS"
            },
            "body": json.dumps("CORS preflight success")
        }

    # -------- 2. READ REQUEST BODY --------
    try:
        body = json.loads(event.get("body", "{}"))
    except Exception:
        body = {}

    # Extract critical fields
    input_pan = body.get("pan", "").strip()
    input_mobile = body.get("mobile", "").strip()
    address_data = body.get("address", {})
    
    # Extract & Validate Income (Backend Enforcement)
    try:
        annual_income = float(body.get("annualIncome", 0))
    except ValueError:
        annual_income = 0
        
    if annual_income < 300000:
        return {
            "statusCode": 400,
            "headers": {
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Headers": "Content-Type"
            },
            "body": json.dumps({"message": "Application rejected: Annual income must be at least 3,00,000."})
        }

    # -------- 3. DUPLICATE CHECK --------
    table = dynamodb.Table(TABLE_NAME)
    
    try:
        # Check if PAN or Mobile already exists
        response = table.scan(
            FilterExpression=Attr('pan').eq(input_pan) | Attr('mobile').eq(input_mobile),
            ProjectionExpression="pan, mobile"
        )
        
        if response.get('Count', 0) > 0:
            return {
                "statusCode": 409, # Conflict
                "headers": {
                    "Access-Control-Allow-Origin": "*",
                    "Access-Control-Allow-Headers": "Content-Type"
                },
                "body": json.dumps({
                    "message": "Application rejected: A user with this PAN or Mobile number already exists."
                })
            }
            
    except Exception as e:
        print(f"Validation Error: {str(e)}")
        # If DB read fails, we generally return 500
        return {
            "statusCode": 500,
            "headers": {"Access-Control-Allow-Origin": "*"},
            "body": json.dumps({"message": "Internal Validation Error"})
        }

    # -------- 4. PREPARE ITEM --------
    application_id = str(uuid.uuid4())
    
    item = {
        "applicationId": application_id,
        "timestamp": datetime.now().isoformat(),
        
        # Personal Info
        "name": body.get("name", ""),
        "email": body.get("email", ""),
        "dob": body.get("dob", ""),
        "countryCode": body.get("countryCode", ""),
        "mobile": input_mobile,
        "pan": input_pan,
        "annualIncome": str(annual_income), # Store as string or number based on preference

        # Address Info
        "doorNo": address_data.get("doorNo", ""),
        "street": address_data.get("street", ""),
        "city": address_data.get("city", ""),
        "district": address_data.get("district", ""),
        "state": address_data.get("state", "Tamil Nadu"),
        "postalCode": address_data.get("postalCode", ""),

        # Loan Info
        "loanAmount": body.get("loanAmount", ""),
        "loanType": body.get("loanType", "")
    }

    # -------- 5. STORE IN DYNAMODB --------
    try:
        table.put_item(Item=item)
    except Exception as e:
        print(f"DynamoDB Write Error: {str(e)}")
        return {
            "statusCode": 500,
            "headers": {
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Headers": "Content-Type"
            },
            "body": json.dumps({"message": "Database save failed", "error": str(e)})
        }

    # -------- 6. SEND EMAIL --------
    try:
        ses.send_email(
            Source=SENDER_EMAIL,
            Destination={"ToAddresses": [ADMIN_EMAIL]},
            Message={
                "Subject": {"Data": f"New Loan Application: {item['name']}"},
                "Body": {
                    "Text": {
                        "Data": f"""
New Loan Application Received

-- Personal Details --
Name: {item['name']}
Email: {item['email']}
DOB: {item['dob']}
Mobile: {item['countryCode']} {item['mobile']}
PAN Number: {item['pan']}
Annual Income: Rs. {item['annualIncome']}

-- Address --
{item['doorNo']}, {item['street']}
{item['city']}, {item['district']}
{item['state']} - {item['postalCode']}

-- Loan Request --
Loan Type: {item['loanType']}
Amount: Rs. {item['loanAmount']}

-- System --
Application ID: {application_id}
Submitted At: {item['timestamp']}
"""
                    }
                }
            }
        )
    except Exception as e:
        print(f"SES Error: {str(e)}")

    # -------- 7. FINAL RESPONSE --------
    return {
        "statusCode": 200,
        "headers": {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Headers": "Content-Type",
            "Access-Control-Allow-Methods": "POST,OPTIONS"
        },
        "body": json.dumps({
            "message": "Loan application submitted successfully",
            "applicationId": application_id
        })
    }
