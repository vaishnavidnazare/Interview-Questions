1. What is an AWS SCP?
Ans: 
    SCP stands for Service Control Policy, It used for restricting certain actions across a certain AWS accont.
    For example,
    The company wants to prevent anyone from deleting an S3 bucket in any account.
    You can attach an SCP that denies the s3:DeleteBucket action.
    So, even if an IAM allows the deletion of an S3 bucket , the SCP will block it.

    SCP is created and managed through AWS Organizations, and you can attach it at different levels of the organization hierarchy.

    Common SCP use cases
        - Prevent users from disabling CloudTrail.
        - Restrict AWS regions where resources can be created.
        - Prevent deletion of important resources.
        - Block use of certain services.
        - Prevent member accounts from leaving the organization.
        - Enforce organization-wide security guardrails.

2. What is AWS Organizations?
Ans: 
    AWS Organizations is a service that lets you manage multiple AWS accounts from one central place.
    What can you do with AWS Organizations?
    - Centralized billing: Combine eligible usage and billing across accounts.
    - Group accounts: Organize accounts into organizational units (OUs).
    - Apply security restrictions: Use SCPs to restrict what accounts can do.
    - Manage multiple accounts: Apply organization-level policies and governance.

    Simple example
    Your company has separate Dev and Production accounts. You want to prevent both teams from using an unapproved AWS region.
    You can use AWS Organizations with SCPs to apply that restriction across both accounts.


3. What is AWS Control Tower?
Ans:
    AWS Control Tower helps you set up and manage a secure, standardized environment for multiple AWS accounts.
    Control Tower helps set up a governed AWS environment called a landing zone.

4. IAM
   
   Role = trust policy + permissions policy. The trust policy says who can assume the role. The permissions policy says what the role can do.

5. How to login and access AWS?
Ans:
    SSO:
    SSO stands for Single Sign-On. In AWS, the current service name is AWS IAM Identity Center, previously called AWS SSO.
    In simple words, it lets you sign in once using your company identity and access the AWS accounts and roles assigned to you.

    Command	                                Purpose
    aws configure	                    Traditionally configures access keys and default CLI settings
    aws configure sso	                Configures access through IAM Identity Center
    aws sso login	                    Authenticates the configured SSO session
    aws sts get-caller-identity	        Shows which identity your CLI is currently using

    Two Imp files are the 
    ~/.aws/credentials -> This file stores credentials such as access key IDs and secret access keys. 

        [default]
        aws_access_key_id = EXAMPLEACCESSKEY
        aws_secret_access_key = EXAMPLESECRETKEY

        [dev]
        aws_access_key_id = EXAMPLEDEVKEY
        aws_secret_access_key = EXAMPLEDEVSECRET

    ~/.aws/config -> This file stores CLI settings and profile configuration.

        [default]
        region = ap-south-1
        output = json

        [profile dev]
        region = ap-south-1
        output = json

        [profile staging]
        region = us-east-1
        output = yaml

    Example of an SSO profile, 

        [profile dev]
        sso_session = company
        sso_account_id = 111122223333
        sso_role_name = DeveloperAccess
        region = ap-south-1
        output = json

        [sso-session company]
        sso_start_url = https://example.awsapps.com/start
        sso_region = ap-south-1
        sso_registration_scopes = sso:account:access
    
    Notice that the profile does not need to contain an access key or secret key. Instead, the CLI uses the SSO session to obtain temporary credentials.


    STS - Security Tokn Service.
    AWS STS helps you obtain temporary credentials for AWS access.
    



   

    