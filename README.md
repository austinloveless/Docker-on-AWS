# AWS Fargate: Running a serverless Node.js app on AWS ECS. 

## Project Summary 

We're going to containerize a node.js project that renders a simple static site and deploy it to an Amazon ECS Fargate cluster. I will supply all the code at https://github.com/austinloveless/Docker-on-AWS. 

The Tutorial assumes you have a basic understanding of Docker. If you are brand new to to Docker you can read my previous [post](https://medium.com/@awsmeetupgroup/docker-on-aws-1855b825de5e) introducing you to Docker.

## Installing Prerequisites 

### Downloading Docker Desktop. 

If you are on a Mac go to https://docs.docker.com/docker-for-mac/install/ or Windows go to https://docs.docker.com/docker-for-windows/install/. Follow the installation instructions and account setup. 

### Installing node.js

Download node.js [here](https://nodejs.org/en/). 

### Installing the AWS CLI

Follow the instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).

## Project setup

Now that we have our prerequisites installed we can build our application. This project isn't going to focus on the application code, the point is to get more familiar with Docker and AWS. So you can download the [Repo](https://github.com/austinloveless/Docker-on-AWS) and change directories into the `Docker-on-AWS` directory. 

If you wanted to run the app locally and say screw docker you can run `npm install` in side the `Docker-on-AWS` directory. Then run `node app.js`. To see the site running locally visit `http://localhost:80`.

Now that we have docker installed and the repo downloaded we can look at the `Dockerfile`. You can think of it as a list of instructions for docker to execute when building a container or the blueprints for the application. 

```
FROM node:12.4-alpine

RUN mkdir /app
WORKDIR /app

COPY package.json package.json
RUN npm install && mv node_modules /node_modules

COPY . .

LABEL maintainer="Austin Loveless"

CMD node app.js

```

At the top we are declaring our runtime which is `node:12.4-alpine`. This is basically our starting point for the application. We're grabbing this base image "FROM" the official docker hub [node image](https://hub.docker.com/_/node).

If you go to the link you can see 12.4-alpine. The "-alpine" is a much smaller base image and is recommended by docker hub "when final image size being as small as possible is desired". Our application is very small so we're going to use an alpine image. 

Next in the Dockerfile we're creating an `/app` directory and setting our working directory within the docker container to run in `/app`. 

After that we're going to "COPY" the `package.json` file to `package.json` on the docker container. We then install our dependencies from our `node_modules`. "COPY" the entire directory and run the command `node app.js` to start the node app within the docker container. 

## Using Docker

Now that we've gone over the boring details of a Dockerfile, lets actually build the thing. 

So when you installed Docker Desktop it comes with a few tools. Docker Command Line, Docker Compose and Docker Notary command line. 

We're going to use the Docker CLI to:

- Build a docker image

- Run the container locally

### Building an image

The command for building an image is `docker build [OPTIONS] PATH | URL | -`. You can go to the [docs](https://docs.docker.com/engine/reference/commandline/build/) to see all the options. 

In the root directory of the application you can run `docker build -t docker-on-aws .`. This will tag our image as "docker-on-aws". 

To verify you successfully created the image you can run `docker images`. Mine looks like `docker-on-aws                                                                  latest              aa68c5e51a8e        About a minute ago   82.8MB`. 

### Running a container locally

Now we are going to run our newly created image and see docker in action. Run `docker run -p 80:80 docker-on-aws`. The `-p` is defining what port you want your application running on. 

![dockerrun](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/Screen+Shot+2020-01-13+at+6.06.33+PM.png)

You can now visit http://localhost:80. 

To see if your container is running via the CLI you can open up another terminal window and run `docker container ls`. To stop the image you can run `docker container stop <CONTAINER ID> `. Verify it stopped with `docker container ls` again or `docker ps`. 

## Docker on Amazon ECS

We're going to push the image we just created to Amazon ECR, Elastic Container Registry, create an ECS cluster and download the image from ECR onto the ECS cluster. 

Before we can do any of that we need to create an IAM user and setup our AWS CLI. 


### Configuring the AWS CLI

We're going to build everything with the AWS CLI. 

Go to the AWS Console and search for IAM. Then go to "Users" and click the blue button "Add User".

![Add User](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/IAMAddUser.png)

Create a user name like "ECS-User" and select "Programmatic Access". 

![Programmatic Access](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/ProgrammaticAccess.png)

Click "Next: Permissions" and select "Attach exisiting policies directly" at the top right. Then you should see "AdministratorAccess", we're keeping this simple and giving admin access. 

Click "Next: Tags" and then "Next: Review", we're not going to add any tags, and "Create user". 

Now you should see a success page and an "Access key ID" and a "Secret access key". 

![Access Key](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/AccessKey.png)

Take note of both the Access Key ID and Secret Access key. We're going to need that to configure the AWS CLI. 

Open up a new terminal window and type `aws configure` and input the keys when prompted. Set your region as `us-east-1`.

![AWS Configure](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/AWSConfigure.png)

### Creating an ECS Cluster

To create an ECS Cluster you can run the command `aws ecs create-cluster --cluster-name docker-on-aws`. 

We can validate that our cluster is created by running `aws ecs list-clusters`. 

![Create Cluster](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/ECSCreate.png). 

If you wanted to delete the cluster you can run `aws ecs delete-cluster --cluster docker-on-aws`


### Pushing an Image to Amazon ECR

Now that the CLI is configured we can tag our docker image and upload it to ECR. 

First, we need to login to ECR. 

Run the command `aws ecr get-login --no-include-email`. The output should be `docker login -u AWS -p` followed by a token that is valid for 12 hours. Copy and run that command as well. This will authenticate you with Amazon ECR. If successful you should see "Login Succeeded". 

Create an ECR Repository by running `aws ecr create-repository --repository-name docker-on-aws/nodejs`. That's the cluster name followed by the image name. Take note of the `repositoryUri` in the output.

![Create repository](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/CreateRepo.png)

We have to tag our image so we can push it up to ECR. 

Run the command `docker tag docker-on-aws <ACCOUNT ID>.dkr.ecr.us-east-1.amazonaws.com/docker-on-aws/nodejs`. Verify you tagged it correctly with `docker images`. 
 
Now push the image to your ECR repo. Run `docker push <ACCOUNT ID>.dkr.ecr.us-east-1.amazonaws.com/docker-on-aws/nodejs`. Verify you pushed the image with `aws ecr list-images --repository-name docker-on-aws/nodejs`. 

![Verify Image](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/ecrList.png)


## Uploading a node.js app to ECS

The last few steps involve pushing our node.js app to the ECS cluster. To do that we need to create and run a task definition and a service.
Before we can do that we need to create an IAM role to allow us access to ECS.

### Creating an ecsTaskExecutionRole with the AWS CLI

I have created a file called `task-execution-assume-role.json` that we will use to create the ecsTaskExecutionRole from the CLI. 

```{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "ecs-tasks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

You can run `aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json ` to create the role. Take note of the `"Arn"` in the output. 
 
![Iam](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/IamCreateRoleCLI.png)

Then run `aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy` to attach the "AmazonECSTaskExecutionRolePolicy".

Take the `"Arn"` you copied earlier and paste it into the  `node-task-definition.json` file for the `executionRoleArn`.

```
{
    "family": "nodejs-fargate-task",
    "networkMode": "awsvpc",
    "executionRoleArn": "arn:aws:iam::xxxxx:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "nodejs-app",
            "image": "xxxxx.dkr.ecr.us-east-1.amazonaws.com/docker-on-aws/nodejs:latest",
            "portMappings": [
                {
                    "containerPort": 80,
                    "hostPort": 80,
                    "protocol": "tcp"
                }
            ],
            "essential": true
        }
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "256",
    "memory": "512"
}
```

### Registering an ECS Task Definition

Once your IAM role is created and you updated the `node-task-definition.json` file with your `repositoryUri` and `executionRoleArn` and  you can register your task. 

Run `aws ecs register-task-definition --cli-input-json file://node-task-definition.json `

### Creating and ECS Service    

The final step to this process is creating a service that will run our task on the ECS Cluster. 

We need to create a security group with port 80 open and we need a list of public subnets for our network configuration. 

To create the security group run `aws ec2 create-security-group --group-name ecs-security-group --description "Security Group us-east-1 for ECS"`. That will output a security group ID. Take note of this ID.  You can see information about the security group by running 
`aws ec2 describe-security-groups --group-id <YOUR SG ID>`. 

It will show that we don't have any IpPermissions so we need to add one to allow port 80 for our node application. Run `aws ec2 authorize-security-group-ingress --group-id <YOUR SG ID> --protocol tcp --port 80 --cidr 0.0.0.0/0` to add port 80. 

Now we need to get a list of our public subnets and then we can create the ECS Service. 

Run `aws ec2 describe-subnets` in the output you should see `"SubnetArn"` for all the subnets. At the end of that line you see "subnet-XXXXXX" take note of those subnets. Note: if you are in `us-east-1` you should have 6 subnets

Finally we can create our service. 

Replace the subnets and security group Id with yours and run ` aws ecs create-service --cluster docker-on-aws --service-name nodejs-service --task-definition nodejs-fargate-task:1 --desired-count 1 --network-configuration "awsvpcConfiguration={subnets=[ subnet-XXXXXXXXXX,
subnet-XXXXXXXXXX,
subnet-XXXXXXXXXX,
subnet-XXXXXXXXXX,
subnet-XXXXXXXXXX,
subnet-XXXXXXXXXX],securityGroups=[sg-XXXXXXXXXX],assignPublicIp=ENABLED}" --launch-type "FARGATE"`. 

Running this will create the service `nodejs-service` and run the task `nodejs-fargate-task:1`. The `:1` is the revision count. When you update the task definition the revision count will go up. 

## Viewing your nodejs application. 

Now that you have everything configured and running it's time to view the application in the browser. 

To view the application we need to get the public IP address. Go to the ECS dashboard, in the AWS Console, and click on your cluster.

![ClusterConsole](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/Screen+Shot+2020-01-13+at+5.52.10+PM.png)

Then click the "tasks" tab and click your task ID. 

![Tasks](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/Tasks.png)

From there you should see a network section and the "Public IP". 

![NetworkConfig](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/Screen+Shot+2020-01-13+at+5.54.07+PM.png)


Paste the IP address in the browser and you can see the node application. 

![nodeApp](https://awsmeetupgroupreadmeimages.s3.amazonaws.com/Docker-on-AWS/NodeApp.png)

Bam! We have a simple node application running in an Amazon ECS cluster powered by Fargate. 

If you don't want to use AWS and just want to learn how to use Docker check out my last [blog](https://medium.com/@awsmeetupgroup/tutorial-docker-and-node-js-2d7fde6eb38b)

Also, I attached some links here for more examples of task definitions you could use for other applications. 

https://docs.aws.amazon.com/AmazonECS/latest/developerguide/example_task_definitions.html

https://github.com/aws-samples/aws-containers-task-definitions/blob/master/
