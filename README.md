# Deployment of Moodle on AWS with CloudFormation


With this project you can provision an instance of the e-learning Moodle application (https://moodle.org/) in a few steps.

The infrastructure will be deployed on AWS Cloud, network is going to have two subnets, and all traffic is going to be routed from a load balancer (AWS ELB), the application is going to be deployed on an ECS cluster and connected to an RDS MySQL instance.

<h2>Requirements</h2>

<ul>
<li>An AWS account where the infrastructure is going to live</li>
<li>Knowledge about the AWS Cli and CloudFormation</li>
<li>Knowledge about Docker and application containerization</li>
</ul>

<h2>Implemented AWS Resources</h2>

<ul>
<li>VPC Networking</li>
<li>Elastic Load Balancer</li>
<li>ECS with Fargate, ECR</li>
<li>Secrets Manager</li>
<li>RDS MySQL</li>
</ul>

<h2>Excecution Steps</h2>

<b>First from the provided Dockerfile, you need to build an image and tag it:</b>

- build -t moodle .

Create an ECR respoitory and push the image you built:

- aws create-repository --repository-name moodle --profile AWS_PROFILE
- aws ecr get-login-password --region AWS_REGION --profile AWS_PROFILE | docker login --username AWS --password-stdin AWS_ACCOUNT_ID.dkr.ecr.AWS_REGION.amazonaws.com
- docker tag moodle:latest AWS_ACCOUNT_ID.dkr.ecr.AWS_REGION.amazonaws.com/moodle:latest
- docker push AWS_ACCOUNT_ID.dkr.ecr.AWS_REGION.amazonaws.com/moodle:latest

<b>Second you need to execute the following cloudformation commands:</b>

- Create the network stack:

aws cloudformation create-stack --stack-name moodle-network \\ \
  --template-body file://moodle-network.yml \\ \
  --capabilities CAPABILITY_IAM \\ \
  --profile AWS_PROFILE

- Create the database stack:

aws cloudformation create-stack --stack-name moodle-database \\ \
  --template-body file://moodle-database.yml \\ \
  --parameters ParameterKey=DBName,ParameterValue=moodle \\ \
      ParameterKey=NetworkStackName,ParameterValue=moodle-network \\ \
      ParameterKey=DBUsername,ParameterValue=moodle \\ \
  --profile AWS_PROFILE

- Create the service stack:

aws cloudformation create-stack --stack-name moodle-service \\ \
  --template-body file://moodle-service.yml \\ \
  --parameters ParameterKey=NetworkStackName,ParameterValue=moodle-network \\ \
      ParameterKey=ServiceName,ParameterValue=moodle-service \\ \ 
      ParameterKey=ImageUrl,ParameterValue=AWS_ACCOUNT_ID.dkr.ecr.AWS_REGION.amazonaws.com/moodle:latest \\ \
      ParameterKey=ContainerPort,ParameterValue=80 \\ \
      ParameterKey=HealthCheckPath,ParameterValue=/ \\ \
      ParameterKey=HealthCheckIntervalSeconds,ParameterValue=90 \\ \
      ParameterKey=DatabaseStackName,ParameterValue=moodle-database \\ \
  --profile AWS_PROFILE

<b>Additionaly you can create a nested stack in the AWS Console, where you can add your templates and define the dependency.</b>

<h2>Production Deployment</h2>

Since you are creating an Elastic Load Balancer, AWS provides a mechanism to have your load balancer behind a secure connection, for production purposes you can create a public certificate for your domain from the AWS Cli:

- aws acm request-certificate --domain-name --profile AWS_PROFILE

The certificate is going to be available after you make the proper DNS validation from your domain administration console, finally add a CNAME entry from your domain administration console to the load balancer public DNS.

Where:

- AWS_ACCOUNT_ID: the AWS account you will use to implement your infrastructure
- AWS_REGION: the AWS region you choose to deploy your services
- AWS_PROFILE: the AWS profile configured in your local environment

IMPORTANT: if you use this procedure for testing purposes, don't forget to delete all the generated stacks (you can use the following commands in order to do it or you can delete them from the AWS Console) to avoid undesirabled costs.

- aws cloudformation delete-stack --stack-name moodle-service --profile AWS_PROFILE
- aws cloudformation delete-stack --stack-name moodle-database --profile AWS_PROFILE
- aws cloudformation delete-stack --stack-name moodle-network --profile AWS_PROFILE