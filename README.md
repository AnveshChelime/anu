

# EC2 AMI Upgrade Workflow Documentation

This document outlines the task of implementing an EC2 AMI upgrade mechanism using AWS resources, including detailed descriptions of the current implementation, completed tasks, and future enhancements.

---

## Task Overview

The goal of this task is to automate the EC2 AMI upgrade process for Auto Scaling Group (ASG) instances and standalone instances. When a new AMI value is updated in AWS Systems Manager Parameter Store, the workflow triggers a series of automated steps to refresh instances in the ASG or handle standalone instances with the updated AMI.

### Workflow Summary

1. **Parameter Update**: Updating the AMI value in `/ami/linux/test` parameter in Parameter Store.
2. **EventBridge Trigger**: A `PutParameter` event is captured in CloudTrail and matched by an EventBridge rule.
3. **Lambda Execution**: The EventBridge rule triggers a Lambda function that performs the instance refresh or standalone instance update.
4. **Instance Refresh**: The Lambda function refreshes instances in ASGs or updates standalone instances, ensuring volumes are detached and re-attached as necessary.
5. **Dynamic Configurations**: The solution supports excluding specific instances and ASGs, and accommodates dynamic ASG configurations.

---

## Completed Tasks

### 1. Parameter Store and EventBridge Integration

- **Parameter Store**: AMI details are stored in the Parameter Store path `/ami/linux/test`.
- **EventBridge Rule**: Configured to match `PutParameter` events from CloudTrail.
- **Event Pattern**:
  ```json
  {
    "source": ["aws.ssm"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventName": ["PutParameter"],
      "requestParameters": {
        "name": ["/ami/linux/test"]
      }
    }
  }
  ```
- **Outcome**: Successfully triggers Lambda on parameter updates.

### 2. Lambda Function for ASG and Standalone Instance Updates

- **Logic**: The Lambda function:
  - Starts an instance refresh for ASGs using the Auto Scaling API.
  - Handles standalone instances by terminating and launching new instances with the updated AMI.
  - Excludes specific ASGs and instances based on dynamic configurations.
  - Ensures attached volumes are detached and re-attached to new instances.
- **Modules Integrated**:
  - **Alias Module**: Used to dynamically manage Lambda function versions and ensure seamless updates.
  - **IAM Module**: Configured Lambda execution role with necessary permissions.
  - **Tags Module**: Applied tags for resource tracking.
  - **ASG Configuration**: Supports dynamic ASG settings via environment variables.

### 3. Lambda Module Environment Variables

- **Key Environment Variables**:
  - `EXCLUDED_INSTANCES`: List of instances to exclude.
  - `EXCLUDED_ASGS`: List of ASGs to exclude.
  - `ASG_CONFIGURATIONS`: Dynamic configurations for ASGs to adjust refresh behavior.

### 4. Terraform Implementation

- Modularized the Terraform code:
  - **Lambda Module**: Deploys the Lambda function with environment variables.
  - **EventBridge Module**: Creates the EventBridge rule to trigger Lambda.
  - **IAM Module**: Manages permissions for Lambda and other resources.
  - **Lambda Permission**: Ensures EventBridge can invoke Lambda using the correct alias and version.
  - **CloudWatch Event Target**: Configures the target to invoke the Lambda function.

---

## Future Enhancements

1. **Enhanced Monitoring and Logging**:

   - Integrate AWS CloudWatch Alarms to monitor instance refresh failures.
   - Add more detailed logs for troubleshooting.

2. **Dynamic Resource Exclusions**:

   - Implement a centralized mechanism for updating excluded instances and ASGs dynamically.

3. **Modular Expansion**:

   - Extend support for multiple environments (e.g., production, staging) with parameterized configurations.

4. **Standalone Instance Scaling**:

   - Enhance logic for scaling standalone instances and handling unique configurations.

5. **Testing Framework**:

   - Add automated tests to validate the instance refresh and standalone instance update workflows.

---

## Screenshots and Links

### Screenshots

1. **Parameter Store Update**

2. **EventBridge Rule**

3. **Lambda Execution Logs**

### Repository Link

[GitHub Repository for Terraform Code](insert-repo-link)

### Workspace Link

[Terraform Cloud Workspace](insert-workspace-link)

---

## Module Descriptions

### 1. Alias Module

- **Purpose**: Dynamically manages Lambda function versions, ensuring EventBridge always invokes the correct version or alias (e.g., `dev`).
- **Key Features**:
  - Simplifies Lambda version management.
  - Reduces the need for manual updates to EventBridge and permissions.

### 2. IAM Module

- **Purpose**: Creates a role and policy for the Lambda function.
- **Key Permissions**:
  - `ssm:GetParameter`
  - `autoscaling:StartInstanceRefresh`
  - `ec2:AttachVolume`, `ec2:DetachVolume`
  - `ec2:TerminateInstances`, `ec2:RunInstances`

### 3. Tags Module

- **Purpose**: Applies standardized tags to all resources for governance and cost tracking.

### 4. Lambda Module

- **Purpose**: Deploys the Lambda function to handle the instance refresh and standalone instance update logic.
- **Key Environment Variables**:
  - `EXCLUDED_INSTANCES`: List of instances to exclude.
  - `EXCLUDED_ASGS`: List of ASGs to exclude.
  - `ASG_CONFIGURATIONS`: Dynamic configurations for ASG refresh behavior.

### 5. EventBridge Module

- **Purpose**: Configures the EventBridge rule to capture `PutParameter` events and trigger the Lambda function.

### 6. Lambda Permission Module

- **Purpose**: Grants EventBridge permission to invoke the Lambda function using the appropriate alias and version.

---

## Conclusion

The EC2 AMI upgrade workflow is designed to automate instance refreshes for ASGs and updates for standalone instances efficiently and securely. By leveraging AWS services and Terraform, the solution ensures consistency, scalability, and ease of management. Future enhancements aim to further improve the robustness and flexibility of the workflow.




# Terraform AMI Update Feasibility Testing - Final Report

**Date:** _(Insert Date)_  
**Author:** _(Your Name)_  

## Overview
This document captures the feasibility testing results of automating AMI updates for Terraform-managed EC2 instances via AWS Lambda. The goal was to validate if Lambda can accurately identify Terraform-managed EC2 instances, ensure security constraints, and prevent unintended updates.

---

## EC2 Instance Validation

### Missing tfc_wsid Tag
**Scenario:** An EC2 instance was manually created without the `tfc_wsid` tag.  
**Expected Outcome:** Lambda should ignore this instance.  
**Result:** The instance was not included in the workspace processing, and no update was triggered.  

### Incorrect Workspace ID (Tag Modified)
**Scenario:** The `tfc_wsid` tag on an EC2 instance was modified to an invalid workspace ID.  
**Expected Outcome:** Lambda should recognize the mismatch and not trigger a Terraform update.  
**Result:** The incorrect workspace ID was detected, and the instance was skipped.  

### EC2s Not Managed by Terraform
**Scenario:** A manually created EC2 instance was assigned a `tfc_wsid` tag manually.  
**Expected Outcome:** Lambda should fetch the workspace ID but not trigger Terraform, since the instance was not created by Terraform.  
**Result:** Lambda correctly skipped the instance.  

---

## Security and Account Validation

### Workspace ID Matches Correct AWS Account
**Scenario:** Ensure workspace ID belongs to the correct AWS account before triggering an update.  
**Expected Outcome:** Lambda should trigger updates only for workspaces associated with the current AWS account.  
**Result:** The validation worked correctly, and only workspaces under the correct AWS account were updated.  

### Prevent Unauthorized Updates
**Scenario:** An EC2 instance was tagged with a workspace ID from another AWS account.  
**Expected Outcome:** Lambda should detect the workspace mismatch and prevent execution.  
**Result:** The security check successfully skipped the unauthorized workspace.  

### IAM Permission Restrictions
**Scenario:** The IAM permission `autoscaling:DescribeAutoScalingInstances` was removed from Lambda's role.  
**Expected Outcome:** Lambda should fail with an `AccessDenied` error.  
**Result:** The expected error was received, confirming IAM validation.  

---

## Multiple Workspaces and EC2 Instances

### Multiple EC2 Instances Across Workspaces
**Scenario:** Multiple EC2 instances across different workspaces were tested.  
**Expected Outcome:** Lambda should detect all workspaces and update them individually.  
**Result:** Lambda successfully processed multiple workspaces and triggered Terraform for each.  

### ASG-Based EC2 Instances
**Scenario:** Ensured EC2 instances inside an Auto Scaling Group (ASG) were identified correctly.  
**Expected Outcome:** Lambda should trigger a single Terraform update for ASG-based instances instead of multiple redundant runs.  
**Result:** Lambda successfully detected ASG instances and handled them correctly.  

---

## Handling Edge Cases

### Stale Workspaces (AWS Account Deleted)
**Scenario:** A workspace existed in Terraform Cloud for an AWS account that was deleted.  
**Expected Outcome:** Lambda should not process the workspace since the AWS account no longer exists.  
**Result:** Lambda correctly ignored workspaces from deleted AWS accounts.  

### Workspace Created Manually and Assigned Existing Resources
**Scenario:** A new workspace was created, and an existing EC2 instance was manually assigned to it.  
**Expected Outcome:** Lambda should verify the workspace with Terraform API and not rely solely on EC2 tags.  
**Result:** Lambda skipped the manually assigned EC2 instance.  

### Unexpected Changes to Workspace ID Tags
**Scenario:** The workspace ID tags were modified on multiple EC2 instances.  
**Expected Outcome:** Lambda should log validation failures and skip invalid updates.  
**Result:** Lambda successfully logged and skipped incorrectly tagged instances.  

---

## Key Learnings and Findings

- Lambda should never rely on EC2 tags alone for workspace validation.  
- Terraform API validation is critical to ensure updates occur in the correct workspace.  
- Security validation successfully prevented unauthorized updates across AWS accounts.  
- IAM permission `autoscaling:DescribeAutoScalingInstances` is essential for handling ASG-based instances.  
- Launch template validation is necessary for ASG-based instances to avoid misconfigurations.  

---

## Next Steps for Enterprise Solution

### IAM Role Finalization
- Add necessary permissions for Terraform and AWS services.  

### Terraform Module Implementation
- Convert the Lambda solution into a reusable Terraform module.  

### Audit Logging and Security Enhancements
- Log every workspace validation attempt.  
- Maintain records of security failures for better tracking.  

### Stale Workspace Cleanup
- Implement automated removal of workspaces associated with deleted AWS accounts.  

### Deployment Documentation for Partners
- Prepare a deployment guide for rolling out the solution across teams.  

---

## Final Summary

- Feasibility testing was successfully completed.  
- Lambda correctly identifies and triggers Terraform runs for valid workspaces.  
- Security and IAM validation works as expected.  
- The solution is now ready for enterprise implementation.  

---

## Instructions for Adding to GitHub
1. Save this content as `README.md` in your local repository.  
2. Commit and push the file to your GitHub repository using:  
   ```sh
   git add README.md
   git commit -m "Adding Terraform AMI Update Feasibility Report"
   git push origin main
