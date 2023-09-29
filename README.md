## Application specifications:
- Strapi CMS is to be run on AWS ECS clusters in front of Aurora DB.
- The application has to be run on both production and staging environments isolated from each other.

## Prerequisites:
- AWS-CLI
- Terraform
- Docker

## Architecture:
![Alt text](infrastructure/strapi-archi.png)

## Infrastructure:
###1. Setting AWS Network Resources 
A VPC is created on AWS with IP cidr range - 10.0.0.0/16. 6 subnets are created in the VPC. Public, Private and Database Subnets

VPC resources are modularized under `module/terraform-aws-vpc`, which created NGW, IGW, Subnets, Routes, RTs, EIP 

Security groups: `starapi-ecs-stage-sg`, `strapi-ecs-prod-sg`, `strapi-aurora-sg` are created for the resources individually


###2. Deploying Aurora DB
For the deployment of Strapi CMS, have chosen the Aurora Postgresql DB, Single cluster, Multi-AZ enabled with a reader and writer. Two logical databases are created in the Cluster namely `production` and `stage`

**Steps followed to provision and configure the Aurora Database using terraform. Refer `aurora.tf`, `bastion.tf` and the module `terraform-aws-aurora`**
- Cluster is created with encryption and Multi-AZ enabled 
- two instances are associated with cluster
- provisioned a temporary bastion machine, used `remote-exec`,`userdata` and `shell scipt` to create logical databases such as `production` and `stage`
- owner of the `production` db cannot access `stage` db , vice versa. Roles , grants and revokes are configured in such a way
- Security group is tightly configured to allow traffic from ECS and bastion
- passwords for the DB users are created using Terraform `Random` resource
- then the passwords are stored in `Secret Manager`
- IAM role, policy applied that Prod container can retreive only prod secret similarly same way stagging works
  
###3. Configure Strapi for Aurora
**Steps followed to dockerizing Strapi and pushing to AWS Elastic Container Registry (ECR) . Refer `ecr.tf`**
- cloned the opensource repo from Strapi
- modified the env value with localhost information
- tested the built and deploy locally
- created an ECR repo using terraform under `ecr.tf` file
- used `local-exec` to build the Strapi Docker image and push it to ECR

###4. Deploy Strapi on ECS
**Steps followed to provision, configure and deploy Strapi on AWS Elastic Container Service (ECS) . Refer `ecs.tf`, module `terrafom-aws-ecs`**
- Cluster named `Strapi` is created using terraform
- 2 services are created under the Cluster namely `strapi-prod`,`strapi-stage`
- 2 task definitions are created for the services. Each task definition has a dedicated Security Groups and Cloudwatch Log groups
- created an api token for Strapi for the JWT access and stored that on Secret manager
- Task definitions are configured with reference to ECR image, Secret Manager's Secret, SGs and Target Groups respectively
- For the sec service to access the respective secret from the secret manager the following custom policy has to be attached to the respecticve `ecsTaskExecutionRole`
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
                ],
            "Resource": [
                "arn:aws:secretsmanager:region:aws_account_id:secret:username_value"
            ]
        }
    ]
}
```
- LoadBalancer is attached to the ECS cluster
- Target Groups are of IP type and listening 1337
- Listner port 80 is always redirected to 443
- Listner port 443 is mapped to target group based on the Path Header of the request.
- Similarly rule with route path `/admin/*` and source ip `my public`, access to admin portal is restricted to anyone else. 

###5. Setup DNS
**Steps followed to provision, configure Route53/DNS. Refer `route53.tf`, module `terrafom-aws-route53`**
- I have created the domain names [strapi.demo.dreamteamit.in](strapi.demo.dreamteamit.in) for production and [stagging-strapi.demo.dreamteamit.in](stagging-strapi.demo.dreamteamit.in) for stagging in Route 53
- Two listner rueles are created in the ALB and the traffic is routed based on the DNS.
- If the HTTPS headder is strapi.demo.dreamteamit.in, the traffic is routed to a target group which points to the prod service in the ECS cluster.
- If the HTTPS header is stagging-strapi.demo.dreamteamit.in, the traffis is routed to a target group which points to the stage service in the ECS cluster.
- For all other traffics, the default rule is configured to show static error messge 503.

###6. Monitor the ECS cluster and DB using Cloud Watch
**Steps followed to push the logs to Cloud Watch and set up alarms.**
- In the task definition `strapi-prod`, monitoring is allowed and a log grpup `/ecs/strapi-prod` is created where the ECS container logs are collected. Similarly for `straip-stage`, a log group `/ecs/strapi-stage` is created.
- In order to enable the ecs tasks/services to push logs to cloudwatch, the following permissions are to be added to the IAM role `ecsTaskExecutionRole`
1. `logs:CreateLogStream` permission allows the ecs service to create a log stream for itself.
2. `logs:CreateLogGroup` permission allows the ecs service to create a log group for itself.
3. `logs:PutLogEvents` permission allows the ECS service to send the logs to the CW logstream.
- Alarms are created in Cloud Watch to monotor the real-time values of the metrics such as CPU usage, Memory usage, etc.,
