
# How to Deploy a Node-Express App on Amazon ECS

The purpose of this repository is to demonstrate how to deploy a simple web application built by Express - Node.js web application framework on Amazon ECS by using Amazon ECR (Elastic Container Registry)

## Requirements

 - Dockerfile and the sample code are provided in this repository.
 - In the optional part 1, we'll [install Docker on AWS EC2](https://docs.aws.amazon.com/AmazonECS/latest/userguide/docker-basics.html#install_docker), build and run the image. You might also use local development environment for this part.
 - In the optional part 2, we'll push the image to [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html#cli-authenticate-registry). If you're using the EC2 Instance for this, make sure its assigned role has necessary rights (such as AmazonEC2ContainerRegistryFullAccess )


## Part 1 - Optional : Running Docker on Amazon EC2

- Launch an EC2 instance with Amazon Linux 2.`t3.micro` size, it will sufficient for this demonstration
- We'll use `port 80` for web application, configure your Security Group to allow HTTP access at this port. 
- Once your instance is up and running, SSH into it and Run the commands below on your instance :

```bash
# Install Docker
$ sudo amazon-linux-extras install docker
$ sudo service docker start
# to restart at every reboot
$ sudo chkconfig docker on
# adding the ec2-user to the docker
$ sudo usermod -a -G docker ec2-user
$ sudo reboot
# you'll lose connection to EC2 for a while
# once you're able to SSH again, verify docker installation : 
$ docker info
```

**Note :** In some cases, you may need to reboot your instance to provide permissions for the ec2-user to access the Docker daemon. Try rebooting your instance in that case.

```bash
# install git
$ sudo yum install -y git
$ git clone https://github.com/aws-samples/amazon-ecs-demo-with-node-express
# build docker image
$ cd amazon-ecs-demo-with-node-express/sample-nodejs-app
$ docker build -t sample-nodejs-app .
# verify and get the image id
$ docker images
# run docker image
$ docker run --name dockerized-node-app -p 80:3000 --init --rm sample-nodejs-app

```

Congratulations! Now, you've deployed a containerized Express - Node.js web app on Amazon EC2. Visit to public DNS of your instance to see the application.

Note: Don't forget to clean your resources to prevent any unexpected charge. 

<p align="center">
    <img src="./diagram/public_ip.png" alt="web-application" />
<p>

## Part 2 - Optional : Push an image to Amazon ECR

In this part we will the image to a container registry - Amazon ECR in order to use it in an Amazon ECS task definition. We'll proceed with a private repository in this demo, but public repositories and Docker Hub are also supported.

- From IAM, create a role with relevant access to Amazon ECR. (For instance, a role with "AmazonEC2ContainerRegistryFullAccess") 
- From EC2 Dashboard, click Actions and Modify IAM role under Security tab. 

```bash
# Authenticate to your default registry
# update the region and aws_account_id on the below command
$ aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com

# Once you receive 'Login Succeeded" , you can create your private repo on ECR
# update the region on the below command
$ aws ecr create-repository \
    --repository-name sample-nodejs-app \
    --image-scanning-configuration scanOnPush=true \
    --region <region>

# tag and push your image
# update the region and aws_account_id on the below command
$ docker tag sample-nodejs-app:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/sample-nodejs-app:latest

$ docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/sample-nodejs-app:latest

```

Congratulations! Now, you've pushed this image to a private repository of Amazon ECR. This should now be listed on your Images under Amazon ECR repositories. Copy the Image URI as we'll need it in the next step.

<p align="center">
    <img src="./diagram/ecr_private.png" alt="ecr_image" />
<p>

## Part 3 - Deploying the application on Amazon ECS

### 1. Create a Task Definition from the ECS dashboard:

[A task definition](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html) is required to run Docker containers in Amazon ECS to specify parameters such as "The Docker image to use with each container in your task", "How much CPU and memory to use with each task","The launch type to use", "The Docker networking mode to use for the containers in your task" ...

- **Launch type:** ec2
- **Task Definition Name:** sample-nodejs-app
- **Click "Add Container"** under Container Definitions
  - **Container name:** sample-nodejs-app
  - **Image:**'aws_account_id'.dkr.ecr.'region'.amazonaws.com/sample-nodejs-app:latest
  - **Soft limit:** 256
  - **Port Mapping:** 80(host):3000(container)
  - You might leave other settings as default and click add.

- You might leave other settings as default and click create

You should now be able to see your task definition on the console.

### 2. Create Cluster from ECS dashboard:

[An Amazon ECS cluster](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/clusters.html) is a logical grouping of tasks or services. If you are running tasks or services that use the EC2 launch type, a cluster is also a grouping of container instances.

- **Cluster Template:** EC2 Linux + Networking

- **Cluster Name :** sample-nodejs-app-cluster

- **EC2 instance type:** t3.micro

- **Number of instances:** 1
- **Root EBS Volume Size:** 30 GiB
- **VPC/Subnet/Security group:** You can choose from the existing once but make sure your instance is in a public subnet and HTTP protocol on port 80 is allowed

- You might leave other settings as default and click create

You should now be able to see your cluster details on the dashboard.

### 3. Create Amazon ECS Service :

[An Amazon ECS service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) enables you to run and maintain a specified number of instances of a task definition simultaneously in an Amazon ECS cluster.

<p align="center">
    <img src="./diagram/create_service.png" alt="ecs_service" />
<p>

- **Launch type: EC2
- Service name: sample-nodejs-app-service
- Number of tasks: 1

- You might leave other settings as default, proceed through the next steps and click `Create Service`

Congratulations! Now, you've deployed a Express - Node.js web app on Amazon ECS by using an image from ECR. Visit to public DNS of the instance from your cluster to see the application.

<p align="center">
    <img src="./diagram/cluster_details.png" alt="ecs_service" />
<p>

Note: Don't forget to clean your resources to prevent any unexpected charge. 

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## Azure DevOps CI/CD to ECS

This repository now includes an [azure-pipelines.yml](/home/minhtrinh/amazon-ecs-demo-with-node-express/azure-pipelines.yml) pipeline that:

- triggers on every push to any branch
- builds the Docker image from `sample-nodejs-app`
- pushes the image to Amazon ECR
- registers a new ECS task definition revision
- updates the existing ECS service and waits for it to become stable

### 1. Create the pipeline in Azure DevOps

- In Azure DevOps, create a pipeline from this repository.
- Use the existing `azure-pipelines.yml` file.
- Make sure the pipeline runs on `ubuntu-latest`.

### 2. Add pipeline variables

Add these variables in Azure DevOps Pipeline settings or in a variable group:

- `AWS_ACCESS_KEY_ID` as a secret
- `AWS_SECRET_ACCESS_KEY` as a secret
- `AWS_SESSION_TOKEN` as a secret only if you use temporary credentials
- `AWS_ACCOUNT_ID`
- `AWS_REGION`
- `ECR_REPO`
- `ECS_CLUSTER`
- `ECS_SERVICE`
- `ECS_CONTAINER_NAME`
- `APP_DIR`

Recommended values for the resources created in this demo:

- `AWS_REGION=ap-southeast-2`
- `ECR_REPO=sample-nodejs-app`
- `ECS_CLUSTER=sample-nodejs-app-cluster`
- `ECS_SERVICE=sample-nodejs-app-service`
- `ECS_CONTAINER_NAME=sample-nodejs-app`
- `APP_DIR=sample-nodejs-app`

### 3. Minimum AWS permissions for the pipeline user

The AWS identity used by Azure DevOps needs permission to:

- push images to ECR
- describe and create ECR repositories
- describe ECS services and task definitions
- register task definition revisions
- update ECS services
- pass the IAM roles already referenced by the existing ECS task definition

If your ECS task definition uses `taskRoleArn` or `executionRoleArn`, the pipeline identity also needs `iam:PassRole` for those role ARNs.

### 4. How deployment works

On each push, the pipeline:

1. builds the image from `sample-nodejs-app`
2. tags it with the Azure DevOps commit SHA
3. pushes both `<commit-sha>` and `latest` tags to ECR
4. reads the current task definition used by `sample-nodejs-app-service`
5. replaces the image for the `sample-nodejs-app` container
6. registers a new task definition revision
7. updates the ECS service to that new revision

### 5. Important note about branch pushes

The current pipeline deploys on **every branch push**, because the trigger is set to `*`.

If you want deployments only from `main`, change:

```yaml
trigger:
  branches:
    include:
    - "*"
```

to:

```yaml
trigger:
  - main
```

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

## Latest update : Jan 2023
