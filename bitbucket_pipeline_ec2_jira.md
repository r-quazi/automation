curl :


```

curl -X POST \
  https://api.bitbucket.org/2.0/repositories/devops-try/devops-aws-automation/pipelines/ \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "target": {
      "ref_type": "branch",
      "type": "pipeline_ref_target",
      "ref_name": "main",
      "variables": [
        {
          "key": "REGION",
          "value": "us-east-1"
        },
        {
          "key": "Instance_Id",
          "value": "i-0371ba82703004c9b"
        }
      ]
    },
    "pipeline_name": "start-ec2"
  }'
```
