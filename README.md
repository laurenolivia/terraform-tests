<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Overview](#overview)
  - [Installation Guide](#installation-guide)
    - [Prepare the input configuration](#prepare-the-input-configuration)
  - [TFC/TFE configuration](#tfctfe-configuration)
  - [TLZ Deployment](#tlz-deployment)
  - [AWS SSO Setup](#aws-sso-setup)
  - [Accounts Management](#accounts-management)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Overview

This component invokes [terraform-aws-tlz-deploy-automation](https://github.com/opslab-tlz-master/terraform-aws-tlz-deploy-automation) module deploys the initial tlz infrastructure into the customer's management account.

## Installation Guide

### Prepare the input configuration

See the [config parameters guide](docs/tlz_deployment_config.md) for reference
```
cd tlz-bootstrap/tlz-deploy

# Copy the terraform.auto.tfvars.SAMPLE to terraform.auto.tfvars and update input configuration parameters
cp terraform.auto.tfvars.SAMPLE terraform.auto.tfvars

# Upload Input Configuration to customer VCS
git add terraform.auto.tfvars
git commit -m "tlz bootstrap input configuration"
git push mytlz master
```


## TFC/TFE configuration
- Create a terraform workspace called tlz-bootstrap under terraform organization you created as part of pre-reqs and select Version control workflow
- Associate the tlz-bootstrap repository to the workspace
- Set the **tlz-deploy** working folder for the workspace
- Make sure to select Terraform v13.1 in Terraform Workspace settings
- Associate 'vcs-key-for-tfc-modules' SSH Key to terraform workspace
- Attach the AWS management account IAM credetials (tlz_deploy_user) to the workspace environment variables as below
  - AWS_ACCESS_KEY_ID
  - AWS_SECRET_ACCESS_KEY (sensitive environment variable)
- Add the following tokens as terraform workspace sensitive variables
  - tfe_api_token                # This is your Terraform Cloud User API Token
  - tfe_vcs_oauth_token          # This is your VCS provider OAuth Token in Terraform Cloud
  - vcs_token                    # This is your Version Control Personal Token


## TLZ Deployment

Deployment is multi-step process

- Upon making above changes to the tlz-bootstrap workspace, perform terraform plan and apply to deploy foundational resources in management account that will enable you to deploy TLZ in your environment
- Login to management account and run the stepfunction called 'tlz-deploy-automation' to deploy and/or update TLZ. This will create AWS Org, Foundational OUs and Foundational Accounts etc
- You will receive "AWS Organizations email verification request" upon AWS Org creation, make sure to complete the same
- The stepfunctions execution will fail in bootstrap phase with below KeyID error while account creation is in process
```
[ERROR] KeyError: 'Id'Traceback (most recent call last):  File "/var/task/tlz-bootstrap.py", line 10, in lambda_handler    lambda_handler_inner(event, context)  File "/var/task/tlz-bootstrap.py", line 144, in lambda_handler_inner    deploy_results = create_foundational_accounts(org,config["accounts"],config["org_admin_role"])  File "/var/task/tlz-bootstrap.py", line 72, in create_foundational_accounts    acc = org.create_account(org_account_role, {"id": account["email"], "account_name": account["name"]})  File "/opt/python/lib/python3.7/site-packages/aws_org.py", line 365, in create_account    return self.get_account(acc)  File "/opt/python/lib/python3.7/site-packages/aws_org.py", line 265, in get_account    print(f"    Getting account details for {account['Id']}")
```
This is a known issue and we have a bugTask created to resolve soon. No worries, in the meantime just re-run the stepfunctions to continue the next steps
- The stepfunctions execution will fail in bootstrap phase with below error
```
An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:sts::<management-acctid>:assumed-role/tlz_deploy_role/tlz-bootstrap is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::<landingzone-services-acctid>:role/tlz_organization_account_access_role
```
We need to update lambda to incorporate the logic to add policy automatically. But for now just add below IAM inline policy to 'tlz_deploy_role' in management account and re-run stepfunctions execution
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "0",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::<landingzone-services-acctid>:role/tlz_organization_account_access_role"
            ]
        }
    ]
}
```
- If foundational account baselines or avm changes need to be affected, monitor the respective workspaces as the stepfuction orchestrates the changes

## AWS SSO Setup

Run below steps in Management Account to quickly setup AWS SSO so that you can login to other TLZ AWS Accounts.

- go to AWS Single Sign-On service page and click on 'Enable AWS SSO'
- Click on Users and Add user for yourself, skip creating group
- Click on AWS Accounts, Select Permission sets tab, Click on Create Permission set
- Use and existing job function policy and select AdministratorAccess and click create
- Select AWS Organization tab on the same page and select all the AWS accounts, Click on Assign users
- Select your user name, select AdministratorAccess permission set and complete the process
- You should have received an email for 'Invitation to join AWS Single Sign-On', Accept Invitation, reset your password and login to AWS SSO
- Do not forget to bookmark your AWS SSO URL as we will be using this frequently in following steps


## Accounts Management

Once stepfunctions workflow execution in management account is complete, proceed to [Accounts Management](https://github.com/opslab-tlz-master/tlz-bootstrap/blob/master/accounts-management/README.md) deployment steps.

