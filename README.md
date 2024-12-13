

# AWS EC2 State Management

## Overview
This document outlines the steps to manage the state of AWS EC2 instances through Jira service requests. It is part of the "IT Infra and DevOps" Helpcenter, providing a streamlined process for EC2 state changes such as starting, stopping, rebooting, or terminating instances.

---

### Accessing the Service Request
1. **Navigate to the Helpcenter**: Go to the "IT Infra and DevOps" Helpcenter page ([Link here]).
2. **Locate the DevOps Self-Service Portal**: Within the Helpcenter, find and select the "DevOps Self-Service Portal" group. Here, you will see multiple service requests available, including:
   - **AWS EC2 Instance State Management**
   - **GCP VM State Management**
   - Other related requests

   ![Image: DevOps Self-Service Portal]
   [Link to DevOps Self-Service Portal]

3. **Select AWS EC2 Instance State Management**: From the available service requests, choose the "AWS EC2 Instance State Management" option.

   ![Image: All Service Requests in DevOps Self-Service Portal Group]
   [Link to AWS EC2 Instance State Management Service Request]

---

### Filling the Service Request Form
Upon selecting the "AWS EC2 Instance State Management" request, you will be presented with a form. Fill out the following fields accurately:

#### 1. **Raise This Request On Behalf Of**
   - Enter your email if you are making the request for yourself.
   - Alternatively, enter the email of the person you are making the request on behalf of.

#### 2. **Summary**
   - Provide a concise summary, e.g., *"AWS EC2 - Start"* or *"AWS EC2 - Stop"*.

#### 3. **Description**
   - Clearly state the reason for starting, stopping, rebooting, or terminating the EC2 instance.

#### 4. **Team**
   - Select the team to which you belong.

#### 5. **AWS Account**
   - Choose the AWS account where the EC2 instance is hosted.

#### 6. **AWS Region**
   - Specify the region where the EC2 instance is provisioned.

#### 7. **AWS EC2 Instance ID**
   - Enter the EC2 instance ID. You can find this information in several ways:
     - On a Linux EC2 instance: Run `curl http://169.254.169.254/latest/meta-data/instance-id`
     - On a Windows EC2 instance: Open PowerShell and run `Invoke-RestMethod -Uri http://169.254.169.254/latest/meta-data/instance-id`.
     - Alternatively, check the Confluence page for instance details.

#### 8. **EC2 State**
   - Select the desired action:
     - **Start**: To start the instance
     - **Stop**: To stop the instance
     - **Reboot**: To reboot the instance
     - **Terminate**: To permanently delete the instance

#### 9. **Approver**
   - Enter the email address of the person authorized to approve this request.

---

### Submitting the Request
Once all fields are completed:
1. Submit the service request.
2. The request will be routed to the approver for authorization.
3. After approval, the automation process will execute the specified action on the EC2 instance.


------



---

### Checking Submitted Requests

To check the status of a submitted request:

1. Click on your profile picture in the Jira Helpcenter.
    
2. Select **Requests** from the dropdown menu.
    
    ![Image: Submitted Requests]
    
3. This will open the Requests page. Locate the relevant request and click on it to view the detailed ticket.
    
    ![Image: Request Details]
    

---

### Approving or Denying Requests

If you are an approver, follow these steps:

1. Click on your profile picture in the Jira Helpcenter.
    
2. Select **Approvals** from the dropdown menu.
    
    ![Image: Approvals Page]
    
3. This will open the Approvals page. Locate the relevant request and click on it to view the ticket in detail.
    
4. Use the **Approve** or **Deny** button to take the necessary action.
    
    ![Image: Approval Actions]



----------


### Making Changes to a Service Request

To modify an already submitted service request:

1. Click on the **Service Request ID** in the Requests page. This will open the corresponding Jira ticket.
    
    ![Image: Service Request Details]
    
2. Update the necessary fields, such as:
    
    - Instance ID
        
    - Region
        
    - Any other details required for the request
        
    
    ![Image: Jira Ticket Edit Fields]
    
3. Save your changes to update the ticket.
    

For additional guidance, consult the Helpcenter or contact the IT Infra and DevOps team.
