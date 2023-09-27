## Apploication specifications:
- Strapi CMS is to be run on AWS ECS clusters in front of Aurora DB.
- The application has to be run on both production and staging environments isolated from each other.

## Prerequisites:
- AWS-CLI
- Terraform
- Docker

## Infrastructure:

A VPC is created on AWS with IP cider range - 10.0.0.0/16. 6 subnets are created in the VPC. The Subnets are partitioned as listed below.
### Availability Zone a:
- Public.strapi-default - cider range 10.0.0.0.0/22
- Private.strapi-default - cider range 10.0.64.0/22
- Private.database.strapi-default - cider range 10.0.128.0/24
### Availability Zone b:
- Public.strapi-default - cider range 10.0.0.0.0/22
- Private.strapi-default - cider range 10.0.64.0/22
- Private.database.strapi-default - cider range 10.0.128.0/24

All the IP ranges are invoked from the terraform variable file tf.vars

The strapi ems application is dockerized and puched to the Elastic Container Registry using terraform.

An EC2 instance with public IP 44.208.33.189/32 is run temporarily as a bastion host in the public.strapi-default in az-a in order to allow automate the DB configuraions. The security group of this bastion host is configured to allow inboubnd traffic only from the public IP of our home network. Also the security group of the bastion host is whitelisted in all other security groups. Post deployment the bastion host can be terminated.

The Aurora DB postgresql cluster is placed in the subnet group from both the availability zones in order to achieve high availability. The security groups for the DB are configured in such a way to allow inboubd traffic only from the private.strapi-default subnets from both the availability zones in order to allow the application tier to communicate with the DB.

In a Elastic Container Service (ECS) cluster, the services for production and staging environments are running in two different subnets. The traffic between these two environments are isolated by using an Application Load Balancer which is provisioned in the public subnet.

In order to limit the access to the databases the credentials are stored in the AWS Secrets Manager and passed as environment variable to the ECS task definitions. IAM roles are created and attached to the ECS task definitions in such a way that the prod service can access only the credentials for prod DB and staging services can access only the credentials for staging DB from the secrets manager.

Using Route 53, the URLs [strapi.dreamteamit.in](strapi.dreamteamit.in) and [stage-strapi.dreamteamit.in](stage-strapi.dreamteamit.in) are created and the traffics hitting these URLs are routed using the listener rules in the ALB as follows:
- If the HTTPS headder is strapi.dreamteamit.in, the traffic is routed to a target group which points to the prod service in the ECS cluster.
- If the HTTPS header is stage-strapi.dreamteamit.in, the traffis is routed to a target group which points to the stage service in the ECS cluster.
- For all other traffics, the default rule is configured to show static error messge 503.

In order to monitor the ECS and Database health, the logs are captured using Cloud Watch and alarms are configured for the critical metrics such as CPU and Memory.
