Lambda code that will gather all ec metadata and update the confluence page :
```python

import boto3
import json
import base64
import os
from datetime import datetime
from urllib import request, parse, error

CONFLUENCE_API_URL = os.environ.get("CONFLUENCE_API_URL")
PAGE_ID = os.environ.get("PAGE_ID")
CONFLUENCE_API_TOKEN = os.environ.get("CONFLUENCE_API_TOKEN")
CONFLUENCE_EMAIL = os.environ.get("CONFLUENCE_EMAIL")


def fetch_all_ec2_instances():
    ec2_data = []
    client = boto3.client("ec2")
    regions = [region["RegionName"] for region in client.describe_regions()["Regions"]]

    for region in regions:
        ec2 = boto3.client("ec2", region_name=region)
        reservations = ec2.describe_instances()["Reservations"]
        for reservation in reservations:
            for instance in reservation["Instances"]:
                ec2_data.append({
                    "Region": region,
                    "AvailabilityZone": instance.get("Placement", {}).get("AvailabilityZone", ""),
                    "InstanceId": instance.get("InstanceId", ""),
                    "InstanceType": instance.get("InstanceType", ""),
                    "State": instance.get("State", {}).get("Name", ""),
                    "AMI": instance.get("ImageId", ""),
                    "PrivateIP": instance.get("PrivateIpAddress", "N/A"),
                    "PublicIP": instance.get("PublicIpAddress", "N/A"),
                    "VPC": instance.get("VpcId", "N/A"),
                    "Name": next(
                        (tag["Value"] for tag in instance.get("Tags", []) if tag["Key"] == "Name"), 
                        "N/A"
                    ),
                    "Tags": {tag["Key"]: tag["Value"] for tag in instance.get("Tags", [])},
                })
    return ec2_data

def generate_html_table(ec2_data):
    """Generates an HTML table from EC2 instance data."""
    table_rows = "".join(
        f"""
        <tr>
            <td>{entry['Region']}</td>
            <td>{entry['AvailabilityZone']}</td>
            <td>{entry['Name']}</td>
            <td>{entry['InstanceType']}</td>
            <td>{entry['State']}</td>
            <td>{entry['AMI']}</td>
            <td>{entry['InstanceId']}</td>
            <td>{entry['PrivateIP']}</td>
            <td>{entry['PublicIP']}</td>
            <td>{entry['VPC']}</td>
            <td>{', '.join(f'{key}: {value}' for key, value in entry['Tags'].items())}</td>
        </tr>
        """
        for entry in ec2_data
    )
    return f"""
    <table border="1" style="border-collapse:collapse; text-align:left; width:100%;">
        <thead>
            <tr>
                <th>Region</th>
                <th>Availability Zone</th>
                <th>Name</th>
                <th>Instance Type</th>
                <th>State</th>
                <th>AMI</th>
                <th>Instance ID</th>
                <th>Private IP</th>
                <th>Public IP</th>
                <th>VPC</th>
                <th>Tags</th>
            </tr>
        </thead>
        <tbody>
            {table_rows}
        </tbody>
    </table>
    """

def update_confluence_page(content):
    """Update Confluence page with the given content."""
    auth_header = base64.b64encode(f"{CONFLUENCE_EMAIL}:{CONFLUENCE_API_TOKEN}".encode()).decode()

    headers = {
        "Authorization": f"Basic {auth_header}",
        "Content-Type": "application/json",
    }
    try:
        fetch_url = f"{CONFLUENCE_API_URL}{PAGE_ID}"
        fetch_request = request.Request(fetch_url, headers=headers)
        with request.urlopen(fetch_request) as response:
            page_data = json.loads(response.read().decode())
        
        version = page_data["version"]["number"] + 1
        
        updated_content = {
            "version": {"number": version},
            "title": page_data["title"],
            "type": "page",
            "body": {
                "storage": {
                    "value": content,
                    "representation": "storage"
                }
            }
        }
        
        update_request = request.Request(
            fetch_url,
            data=json.dumps(updated_content).encode(),
            headers=headers,
            method="PUT"
        )
        with request.urlopen(update_request) as update_response:
            return "Confluence page updated successfully."
    except error.HTTPError as e:
        error_body = e.read().decode() if hasattr(e, 'read') else ''
        return f"Failed to update Confluence page: {e.code}, {e.reason}, Details: {error_body}"
    except Exception as e:
        return f"Error occurred: {str(e)}"

def lambda_handler(event, context):
    ec2_data = fetch_all_ec2_instances()
    
    regions = set(entry['Region'] for entry in ec2_data)
    regions_with_ec2 = set(entry['Region'] for entry in ec2_data if entry['InstanceId'])
    ec2_with_public_ip = sum(1 for entry in ec2_data if entry['PublicIP'] != "N/A")
    ec2_per_vpc = {}
    
    for entry in ec2_data:
        vpc_id = entry['VPC']
        if vpc_id not in ec2_per_vpc:
            ec2_per_vpc[vpc_id] = 0
        ec2_per_vpc[vpc_id] += 1
    
    summary_cards = f"""
    <div style="display: flex; flex-wrap: wrap; gap: 20px; margin: 20px 0;">
        <div style="flex: 1; min-width: 200px; padding: 20px; background: #f8f9fa; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
            <h3 style="margin-top: 0; color: #333;">📊 Overview</h3>
            <p>AWS Account: {boto3.client('sts').get_caller_identity()['Account']}</p>
            <p>Total EC2 Instances: {len(ec2_data)}</p>
            <p>Active Regions: {len(regions_with_ec2)}</p>
        </div>
        <div style="flex: 1; min-width: 200px; padding: 20px; background: #f8f9fa; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
            <h3 style="margin-top: 0; color: #333;">🌐 Network</h3>
            <p>Instances with Public IP: {ec2_with_public_ip}</p>
            <p>Total VPCs: {len(ec2_per_vpc)}</p>
        </div>
    </div>
    """
    
    vpc_summary = """<div style='margin: 20px 0;'><h3>🏢 VPC Distribution</h3>
    <ul style='list-style-type: none; padding: 0;'>"""
    for vpc_id, count in ec2_per_vpc.items():
        vpc_summary += f"<li style='margin: 10px 0; padding: 10px; background: #f8f9fa; border-radius: 4px;'>VPC {vpc_id}: {count} instances</li>"
    vpc_summary += "</ul></div>"
    
    intro_paragraph = """
    <div style="background: #f8f9fa; padding: 20px; border-radius: 8px; margin-bottom: 20px;">
        <h2 style="margin-top: 0;">🖥️ EC2 Instance Overview</h2>
        <p>This page provides an overview of EC2 instances across all AWS regions. 
        The data is automatically updated every 6 hours or when triggered by other Lambda functions.</p>
    </div>
    """
    
    # Use the generate_html_table function to create the table
    table_html = generate_html_table(ec2_data)
    
    timestamp = f"""
    <div style="text-align: right; margin-top: 20px; padding: 10px; background: #f8f9fa; border-radius: 4px;">
        <p style="margin: 0; color: #666;">Last updated by Lambda: {datetime.now().strftime('%Y-%m-%d %H:%M:%S UTC')}</p>
    </div>
    """
    
    content = f"{intro_paragraph}{summary_cards}{vpc_summary}{table_html}{timestamp}"
    
    result = update_confluence_page(content)
    return {"statusCode": 200, "body": result}

```



IAM Permission :

```json

{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"ec2:DescribeInstances",
				"ec2:DescribeTags",
				"ec2:DescribeRegions",
				"ec2:DescribeVpcs",
				"ec2:DescribeAvailabilityZones",
				"ec2:DescribeSecurityGroups",
				"ec2:DescribeKeyPairs",
				"sts:GetCallerIdentity",
				"kms:Decrypt"
			],
			"Resource": "*"
		},
		{
			"Effect": "Deny",
			"NotAction": [
				"ec2:DescribeInstances",
				"ec2:DescribeTags",
				"ec2:DescribeRegions",
				"ec2:DescribeVpcs",
				"ec2:DescribeAvailabilityZones",
				"ec2:DescribeSecurityGroups",
				"ec2:DescribeKeyPairs",
				"sts:GetCallerIdentity",
				"kms:Decrypt"
			],
			"Resource": "*"
		}
	]
}

```
