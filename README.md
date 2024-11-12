# BookStack AWS CloudFormation IaC

## Overview

This repository contains CloudFormation infrastructure as code templates for deploying a 3-tier architecture in AWS for BookStack. 

Provisioned infastructure components:

| Service | Purpose |
| ALB | Load balancing and TLS termination |
| ECS Fargate | Container hosting to run the BookStack application |
| RDS Aurora MySQL | Scalable RDBMS to house BookStack's relational data |
| EFS | Scalable network share to house BookStack's unstructured data such as uploads |
| CW Alarms | Notifications for application availability and performance issues |
| CW DashBoard | Metrics dashboard to monitor and diagnose application health and issues |

## Structure

- **app.yml**: Manages ECS Fargate deployment along with task definitions, roles, auto-scaling configurations, and the file share.
- **database.yml**: Provisions an Aurora MySQL RDS cluster and instance .
- **frontend.yml**: Creates an Application Load Balancer, listeners, and target groups.
- **main.yml**: A parent template used to create nested stacks from the other templates and coordinate passing parameters between them.
- **monitoring.yml**: Sets up CloudWatch alarms and a dashboard for monitoring various aspects of the infrastructure, including ECS, load balancers, and response times.
- **shared-infra.yml**: Defines foundational resources such as security groups that are shared across the entire infrastructure.


## Prerequisites

Before deploying any of these templates, ensure that the following prerequisites are met:

- **AWS Access**: Access to an active AWS account with sufficient permissions to create resources like Stacks, RDS Clusters/Instances, ALBs, IAM roles, Fargate tasks and services, and SSM parameters.

- **AWS CLI**: Install the AWS CLI v2 and configure it using *aws configure*

- **S3 Bucket**: An existing S3 bucket to package and upload the stack templates for deployment


## Deployment

### Step 0: Choose the BookStack domain

You must decide on the domain you will use to point to BooKStack before deployment (e.g. bookstack.mycompany.com)

Because the application is aware of its own URL, it's important to follow BookStack's documentation on updating the domain.

### Step 1: Create an ACM Certificate

You must manually create an ACM certificate that will be used for TLS encryption.  This certificate must include the domain you chose in Step 0.

```bash
aws acm request-certificate --domain-name <YOUR_DOMAIN_NAME> --validation-method DNS
```

After running this command, follow the ACM documentation instructions to validate the domain by creating the DNS validation records. Once validated, note the ACM Certificate ARN for use in subsequent steps.


### Step 2: Setup SSM Parameters 

The *main.yml* template relies on SSM parameters to be provisioned for the VPC, Subnet IDs, and ACM Certificate.

With the ACM certificate ARN you obtained in Step 1, populate the */devops/certificate* parameter:
```bash
aws ssm put-parameter --name "/devops/certificate" --type "String" --value "<YOUR_ACM_CERTIFICATE_ARN>"
```

Get your VPC Id and Subnet Ids
```bash
# List VPC IDs
aws ec2 describe-vpcs --query 'Vpcs[*].VpcId' --output text
# List Subnet IDs
aws ec2 describe-subnets --query 'Subnets[*].SubnetId' --output text
```

Create the following SSM parameters by selecting the appropriate VPC and Subnet IDs to populate into the parameters.
```bash
aws4 ssm put-parameter --name "/devops/web-subnet-1" --type "String" --value "<YOUR_WEB_SUBNET_1>"
aws ssm put-parameter --name "/devops/web-subnet-2" --type "String" --value "<YOUR_WEB_SUBNET_2>"
aws ssm put-parameter --name "/devops/app-subnet-1" --type "String" --value "<YOUR_APP_SUBNET_1>"
aws ssm put-parameter --name "/devops/app-subnet-2" --type "String" --value "<YOUR_APP_SUBNET_2>"
aws ssm put-parameter --name "/devops/data-subnet-1" --type "String" --value "<YOUR_DATA_SUBNET_1>"
aws ssm put-parameter --name "/devops/data-subnet-2" --type "String" --value "<YOUR_DATA_SUBNET_2>"
aws ssm put-parameter --name "/devops/certificate" --type "String" --value "<YOUR_ACM_CERTIFICATE_ARN>"
```
**Note**: The Web, App, and Data subnets will host the load balancer, fargate service, and aurora database respectively.  It's advisable to use 6 different subnets but you can use the same 2 subnets for the web, app, and data.

### Step 3: Package the CloudFormation Templates

Use the AWS CLI to package all nested templates and upload them to S3 in one step using the `package` command.

```bash
# Replace <YOUR_BUCKET_NAME> with the name of your S3 bucket.
aws cloudformation package --template-body file://main.yml --s3-bucket <YOUR_BUCKET_NAME> --output-template-file packaged-main.yml
```

This command will upload all nested template files to your S3 bucket and create a packaged version of `main.yml` that references the S3 locations of the nested templates.

### Step 4: Deploy the Parent Stack

The packaged, parent stack (`packaged-main.yml`) references all other nested stacks through the S3 URLs.  You can use this packaged template to deploy all the infrustructure in the correct order.

You will need to provide parameters like stack name and other relevant settings.

```bash
# Replace <STACK_NAME> with your desired CloudFormation stack name and <YOUR_BUCKET_NAME> with the S3 bucket name.
aws cloudformation create-stack \
  --stack-name <STACK_NAME> \
  --template-body file://packaged-main.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
  ParameterKey=ResourcePrefix,ParameterValue=bookstack \
  ParameterKey=AppDomain,ParameterValue=<YOUR_DOMAIN_NAME> \
  ParameterKey=AppKey,ParameterValue=SomeRand32CharacterKeyEncryption \
  ParameterKey=CreateMonitoring,ParameterValue=false
```

### Step 5: Monitoring the Stack Deployment

Once the deployment starts, you can monitor the progress:

```bash
aws cloudformation describe-stacks --stack-name <STACK_NAME>
aws cloudformation describe-stack-events --stack-name <STACK_NAME>
```

Alternatively, you can use the AWS Management Console to track the status of your CloudFormation stack.

### Step 6: Update the domain

Once the deployment completes successfully, update your domain's DNS to point at the Application Load Balancer's domain.

You can also use the `Outputs` section in the AWS Console for the *frontend* stack to get the Load Balancer ARN.
If you're using Route53 for DNS hosting, follow the instructions @ https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html

## Maintenance

### Updating Resources
- **Rolling Updates**: For ECS and RDS instances, you can use the CloudFormation update functionality to roll out updates without downtime.
- **Parameters Update**: If SSM parameters are updated (e.g., subnet IDs or ARNs), ensure that any dependent stacks are updated accordingly to reflect the new values.

### ACM Certificate Renewal
- The `frontend.yml` creates an ACM certificate for the domain name supplied. The certificate uses DNS validation, and it’s important to ensure that your domain remains properly configured to renew the certificate automatically.

### Monitoring and Alerts
- **CloudWatch Alarms**: The `monitoring.yml` template provisions CloudWatch alarms that can send alerts to the SNS topic specified. Ensure that the SNS topic `/shared-infra/alarm-sns-arn` is subscribed to the appropriate endpoints (e.g., email or Slack) for receiving notifications.
- **CloudWatch Dashboard**: Review the CloudWatch dashboard to monitor infrastructure performance. It provides metrics for ALB response times, ECS task counts, and other KPIs.

## Authors
- Iyad Kandalaft - Primary contibutor