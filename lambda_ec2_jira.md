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

    # Authentication check
    if not validate_token(event):
        return {
            'statusCode': 403,
            'body': json.dumps({
                'error': 'Unauthorized',
                'message': 'Invalid or missing authentication token'
            })
        }

    try:
        # Parse body from the event for Lambda function URL
        body = json.loads(event['body']) if 'body' in event else event

        # Extract parameters
        region = body.get('region')
        instance_ids = body.get('instance_id')
        action = body.get('action')

        # Validate parameters
        if not all([region, instance_ids, action]):
            logger.error("Missing required parameters")
            return {
                'statusCode': 400,
                'body': json.dumps({
                    'error': 'Missing required parameters',
                    'received_data': body
                })
            }

        # Ensure instance_ids is a list, even if the user provides a comma-separated string
        if isinstance(instance_ids, str):
            instance_ids = instance_ids.split(',')

        # Create EC2 client
        ec2 = boto3.client('ec2', region_name=region)

        # Perform action
        if action == 'stop':
            response = ec2.stop_instances(InstanceIds=instance_ids)
            message = f"Stopped instances: {', '.join(instance_ids)}"
        elif action == 'start':
            response = ec2.start_instances(InstanceIds=instance_ids)
            message = f"Started instances: {', '.join(instance_ids)}"
        elif action == 'reboot':
            response = ec2.reboot_instances(InstanceIds=instance_ids)
            message = f"Rebooted instances: {', '.join(instance_ids)}"
        elif action == 'terminate':
            response = ec2.terminate_instances(InstanceIds=instance_ids)
            message = f"Terminated instances: {', '.join(instance_ids)}"
        else:
            logger.error(f"Invalid action: {action}")
            return {
                'statusCode': 400,
                'body': json.dumps({
                    'error': f'Invalid action: {action}',
                    'allowed_actions': ['start', 'stop', 'reboot', 'terminate']
                })
            }

        # Log the action taken
        logger.info(f"{action.capitalize()} instances response: {response}")

        # Return success response
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': message,
                'response': str(response)
            })
        }

    except Exception as e:
        logger.error(f"Error processing request: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': f'Internal server error: {str(e)}',
                'received_data': body if 'body' in locals() else 'No body received'
            })
        }


```

#### Finding IP 
```python

import json
import logging

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def get_requesting_ip(event):
    # Attempt to extract IP from multiple locations
    possible_ips = [
        event.get('requestContext', {}).get('identity', {}).get('sourceIp', ''),
        event.get('headers', {}).get('x-forwarded-for', '').split(',')[0].strip(),
        event.get('requestContext', {}).get('http', {}).get('sourceIp', '')
    ]
    
    # Log all detected IPs for debugging
    logger.info(f"Possible IPs: {possible_ips}")
    
    # Return the first non-empty IP found
    for ip in possible_ips:
        if ip:
            return ip
    
    # If no IP is found, return a default message
    return "IP not found"

def lambda_handler(event, context):
    # Log the full event for debugging
    logger.info(f"Received event: {json.dumps(event)}")

    try:
        # Get the requesting IP
        requesting_ip = get_requesting_ip(event)

        # Log the requesting IP for debugging
        logger.info(f"Requesting IP: {requesting_ip}")

        # Return the IP address in the response body
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json'
            },
            'body': json.dumps({
                'received_ip': requesting_ip
            })
        }

    except Exception as e:
        logger.error(f"Error processing request: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': f'Internal server error: {str(e)}'
            })
        }


```
#### IAM Role for Lambda :
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:RebootInstances",
                "ec2:TerminateInstances"
            ],
            "Resource": [
                "arn:aws:ec2:us-east-1:<account-id>:instance/i-*",
                "arn:aws:ec2:us-west-2:<account-id>:instance/i-*"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": [
                        "us-east-1",
                        "us-west-2"
                    ]
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Deny",
            "Action": [
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:RebootInstances",
                "ec2:TerminateInstances"
            ],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:RequestedRegion": [
                        "us-east-1",
                        "us-west-2"
                    ]
                }
            }
        }
    ]
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
