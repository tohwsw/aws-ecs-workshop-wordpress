

# Getting Started with ECS


## 1. Setting up the VPC

We will create a new VPC for our entire infrastructure. We need 2 public subnets for ECS cluster and the ALB.
Configure a VPC with the following requirements via the CloudFormation script given:

| VPC  |  |
| ------------- | ------------- |
| Name Tag  | ECS Lab VPC  |
| CIDR  | 10.0.0.0/16  |

| subnet a  |   |
| ------------- | ------------- |
| Name tag  | Public subnet a |
| CIDR  | 10.0.0.0/24  |

| subnet b  |   |
| ------------- | ------------- |
| Name tag  | Public subnet b |
| CIDR  | 10.0.1.0/24  |



You can create this with the CloudFormation script using the following link.

Region| Launch
------|-----
Asia Pacific (Singapore) | [![cfn](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/new?stackName=ecsworkshopstack&templateURL=https://s3-ap-southeast-1.amazonaws.com/weitoh/containerworkshop/vpcwizard.json)




## 2. Setting up the IAM user and roles

In order to work with ECS, we will need the appropriate permissions for our ECS cluster. Go to the [IAM Console](https://console.aws.amazon.com/iam/home), Roles -> Create New Role -> AWS Service -> EC2.
In the Create Role screen, enter **AmazonEC2ContainerServiceforEC2Role** **AmazonEC2ContainerServiceAutoscaleRole** in the text field (without a comma) and select the two policies.

![img2]

[img2]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-ecslabinstanceprofile1.png

In the Review screen, enter **ecslabinstanceprofile** for the Role name and click **Create Role**.

![img3]

[img3]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-ecsinstanceprofile2.png

**Note**: By default, the ECS first run wizard creates **ecsInstanceRole** for you to use. However, it's a best practice to create a specific role for your use so that we can add more policies in the future when we need to.

## 3. Launching the Cluster

Next, let’s launch the ECS cluster which will host our container instances. We're going to put these instances in the public subnets since they're going to be hosting public microservices.

Create a new security group by navigating to the EC2 console -> Security Group and create **sgecslabpubliccluster**. Keep the defaults. Make sure the correct VPC is selected when creating the security group.

Navigate to the [ECS console](https://console.aws.amazon.com/ecs/) and click Create Cluster. Choose the **EC2 Linux + Networking** cluster template. Click **Next Step**.

In the next screen, configure the cluster as follows:

| Field Name  | Value |
| ------------- | ------------- |
| Cluster Name  | EcsLabPublicCluster  |
| Provisioning Model  | On-Demand Instance  |
| EC2 instance type  | t2.micro  |
| Number of instances  | 2  |
| EBS storage  | 22  |
| Keypair  | none  |

![img4]

[img4]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-ecslabpubliccluster.png

![img5]

[img5]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-ecslabpubliccluster2.png

Click Create. It will take a few minutes to create the cluster.

## 4. Configuring the AWS CLI

On your laptop, we will use the AWS CLI to create ECS tasks. If AWS CLI has yet to be installed, please bring up a Cloud9 instance by going to https://ap-southeast-1.console.aws.amazon.com/cloud9/home/product. Cloud9 will provide you terminal access to run AWS CLI.

## 5. Create the Task Execution IAM role

Amazon ECS needs permissions so that your Fargate task can store logs in CloudWatch. This permission is covered by the task execution IAM role.

  

Go to [IAM Console](https://console.aws.amazon.com/iam/home), click on **Role** and then **Create Role**

  

Choose **Elastic Container Service** and then **Elastic Container Service Task**

![img6]

[img6]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-taskexecutionrole.png

Next click on **Permissions** and then select **AmazonECSTaskExecutionRolePolicy**

![img7]

[img7]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-taskexecutionrole2.png

Name the role **ecsTaskExecutionRole**

## 6. Create the Task Definitions

On your laptop, we will use the AWS CLI to create ECS task definitions.
Copy the content below and save it as **wordpress.json**. Make sure to change the arn **arn:aws:iam::284245693010:role/ecsTaskExecutionRole** of **ecsTaskExecutionRole** to your own.

```
    {
  "executionRoleArn": "arn:aws:iam::284245693010:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "environment": [
        {
          "name": "WORDPRESS_DB_HOST",
          "value": "mysql-service.ecslab"
        },
        {
          "name": "WORDPRESS_DB_PASSWORD",
          "value": "password"
        }
      ],
      "name": "wordpress",
      "image": "wordpress:4.6",
      "memory": 512,
      "cpu": 256,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fargate",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "wordpress"
        }
      },
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80
        }
      ]
    }
  ],
  "family": "wordpress_fargate_task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "memory": "512",
  "cpu": "256"
}
```
    
Next create mysql.json with the below content. Make sure to change the arn **arn:aws:iam::284245693010:role/ecsTaskExecutionRole** of **ecsTaskExecutionRole** to your own.


```
{
  "executionRoleArn": "arn:aws:iam::284245693010:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "environment": [
        {
          "name": "MYSQL_ROOT_PASSWORD",
          "value": "password"
        }
      ],
      "name": "mysql",
      "image": "mysql:5.7",
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fargate",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "mysql"
        }
      },
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3306,
          "hostPort": 3306
        }
      ]
    }
  ],
  "family": "mysql_fargate_task",
  "memory": "512",
  "cpu": "256",
  "networkMode": "awsvpc",
  "requiresCompatibilities": [
    "FARGATE"
  ]
}
```

Next register the task definitions with ECS. You have to run the commands in the folder containing wordpress.json and mysql.json

    $aws ecs register-task-definition --cli-input-json file://wordpress.json
    
    $aws ecs register-task-definition --cli-input-json file://mysql.json
    
    $aws ecs list-task-definitions

Next, create the CloudWatch log group **/ecs/fargate**. Go to [CloudWatch Console](https://console.aws.amazon.com/cloudwatch). 

![img8]

[img8]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-cloudwatch.png

Give the log group name **/ecs/fargate**

![img9]

[img9]:https://github.com/tohwsw/awsecslab/blob/master/Lab21-Getting-Started-with-ECS/img/1-cloudwatch2.png

## That's a wrap!

You have now created the ECS cluster and the task definitions. You can proceed to lab 2 where the tasks will be run as ECS services.

>Written with [StackEdit](https://stackedit.io/).

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4MTczNzQ2NDUsMTAwMTA3NzUzOCwtNz
MyMTY0NzQ5LDE3OTg3MDYxODYsLTk3NjI2NDQ4Niw0MDEyNjU5
ODEsLTE0ODMzMzg5MzksLTIwOTE2MTQ4MjJdfQ==
-->