### Lambda Code :
```python
import boto3
import json
import logging
import os
import secrets

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Use environment variable for token (more secure than hardcoding)
def validate_token(event):
    # Check headers for authorization
    headers = event.get('headers', {})
    
    # Extract token from Authorization header
    provided_token = headers.get('authorization', '').replace('Bearer ', '')
    
    # Retrieve the expected token from AWS Lambda environment variable
    expected_token = os.environ.get('AUTH_TOKEN')
    
    # Perform constant-time comparison to prevent timing attacks
    if not secrets.compare_digest(provided_token, expected_token):
        logger.error("Unauthorized access attempt")
        return False
    return True

def lambda_handler(event, context):
    # Log the full event for debugging
    logger.info(f"Received event: {json.dumps(event)}")

    # Set CORS headers
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'POST',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        'Content-Type': 'application/json'
    }

    # Handle preflight CORS request
    if event.get('httpMethod') == 'OPTIONS':
        return {
            'statusCode': 200,
            'headers': headers,
            'body': 'Preflight request successful'
        }

    # Authentication check
    if not validate_token(event):
        return {
            'statusCode': 403,
            'headers': headers,
            'body': json.dumps({
                'error': 'Unauthorized',
                'message': 'Invalid or missing authentication token'
            })
        }

    try:
        # Parse body from the event for Lambda function URL
        body = json.loads(event['body']) if 'body' in event else event

        # Log parsed body for verification
        logger.info(f"Parsed body: {json.dumps(body)}")

        # Extract parameters
        region = body.get('region')
        instance_id = body.get('instance_id')
        action = body.get('action')

        # Validate parameters
        if not all([region, instance_id, action]):
            logger.error("Missing required parameters")
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({
                    'error': 'Missing required parameters',
                    'received_data': body
                })
            }

        # Create EC2 client
        ec2 = boto3.client('ec2', region_name=region)

        # Perform action
        if action == 'stop':
            response = ec2.stop_instances(InstanceIds=[instance_id])
            logger.info(f"Stop instances response: {response}")
            return {
                'statusCode': 200,
                'headers': headers,
                'body': json.dumps({
                    'message': f'Instance {instance_id} stopped',
                    'response': str(response)
                })
            }
        elif action == 'start':
            response = ec2.start_instances(InstanceIds=[instance_id])
            logger.info(f"Start instances response: {response}")
            return {
                'statusCode': 200,
                'headers': headers,
                'body': json.dumps({
                    'message': f'Instance {instance_id} started',
                    'response': str(response)
                })
            }
        elif action == 'reboot':
            response = ec2.reboot_instances(InstanceIds=[instance_id])
            logger.info(f"Reboot instances response: {response}")
            return {
                'statusCode': 200,
                'headers': headers,
                'body': json.dumps({
                    'message': f'Instance {instance_id} rebooted',
                    'response': str(response)
                })
            }
        elif action == 'terminate':
            response = ec2.terminate_instances(InstanceIds=[instance_id])
            logger.info(f"Terminate instances response: {response}")
            return {
                'statusCode': 200,
                'headers': headers,
                'body': json.dumps({
                    'message': f'Instance {instance_id} terminated',
                    'response': str(response)
                })
            }
        else:
            logger.error(f"Invalid action: {action}")
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({
                    'error': f'Invalid action: {action}',
                    'allowed_actions': ['start', 'stop', 'reboot', 'terminate']
                })
            }

    except Exception as e:
        logger.error(f"Error processing request: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({
                'error': f'Internal server error: {str(e)}',
                'received_data': body if 'body' in locals() else 'No body received'
            })
        }
```

#### save auth token :
In AWS Lambda Console:

Go to your Lambda function
Navigate to Configuration > Environment variables
Add a new environment variable:

Key: AUTH_TOKEN
Value: A secure, long random string.

#### API Call :
```bash
curl -X POST https://your-lambda-url.lambda-url.us-east-1.on.aws/ \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer your_secure_token_here" \
     -d '{"region": "us-east-1", "instance_id": "i-0d61ec5d5231ab6a4", "action": "stop"}'
```

####  Jira Automation : 
- Web request URL : Your Function URL
- HTTP Method : POST
- Web rewuest body :
-
```
  {
  "region": "us-east-1",
  "instance_id": "i-0371ba82703004c9b",
  "action": "start"
}
```
- Headers :
  - Content-Type : application/json
  - Authorization : Bearer 1234ASDF
