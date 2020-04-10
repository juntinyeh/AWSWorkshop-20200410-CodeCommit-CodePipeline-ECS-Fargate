# AWSWorkshop-20200410-CodeCommit-CodePipeline-ECS-Fargate
An End to End flow from CodeCommit, through CodePipeline to build docker image then push to ECR, then another pipeline will be triggered to deploy the image to ECS Fargate.

------

## In this workshop, we will have a quick review following services:
- Amazon Elastic Container Service (ECS) https://aws.amazon.com/ecs/(https://aws.amazon.com/ecs/)
- Amazon Elastic Container Registry (ECR) https://aws.amazon.com/ecr/(https://aws.amazon.com/ecr/)
- AWS Fargate https://aws.amazon.com/fargate/(https://aws.amazon.com/fargate/)
- AWS CodeCommit https://aws.amazon.com/codecommit/(https://aws.amazon.com/codecommit/)
- AWS CodeBuild https://aws.amazon.com/codebuild/(https://aws.amazon.com/codebuild/)
- AWS CodeDeploy https://aws.amazon.com/codedeploy/(https://aws.amazon.com/codedeploy/)
- AWS CodePipeline https://aws.amazon.com/codepipeline/(https://aws.amazon.com/codepipeline/)
------

### Step 1:
* Switch Region on the AWS console, a drag down menu near right-up corner.
Pick one region close to you, if you don't have any prefer, use **us-east-1**

------
### Step 2: Setup IAM Role/User for this workshop
- **AWS Console > Services > IAM > Role**
- Create Role, service pick "EC2" and click "Next"
- Search "AWSCodeCommitFullAccess" and click the check box
- Click Next, Inut Tag Key and Value if you want
- Click Next to enter "Name" = "Workshop_Cloud9"
- and "Description" = "Temp EC2 Role for Workshop " and click "Create" for the Role.
------

### Step 3: Launch a Cloud9 IDE for our workshop env 
* **AWS Console > Services > Cloud9 > Create Environment**
- Enter "Name" and "Description" for your Environment, and click Next Step
- Select "Create a new instance for environment (EC2)" for Environment Type
- Select "t3.micro" for Instance Type
- Select "Amazon Linux" for Platform, and click Next Step
- Input Tag Key and Value if you want, and click "Create Environment"

***Then we need to Attach the IAM Role onto this Cloud9 Environment***
- **AWS Console > Services > EC2 > Instances > Click the EC2 Instance for your Cloud9 > Actions > Instance Settings > Attach/Replace IAM Role > Choose the IAM Role we created in Step2**
- **Back to Cloud9 > Cloud9 Icon (Upper Left Corner) > Perference > AWS Settings > Credential > AWS Managed Temperory Credential > Off**
------

### Step 4: Setup Go Development Environment in your Cloud9
- After the IDE launch, click to "bash" tab in the botton of the IDE page
- We will follow the AWS Cloud9 User Guide to setup our Go env - https://docs.aws.amazon.com/cloud9/latest/user-guide/sample-go.html(https://docs.aws.amazon.com/cloud9/latest/user-guide/sample-go.html) has all the detail, and we copy the mendatory shell command in below:

```
git clone https://github.com/awshktsa/AWSWorkshop-20200410-CodeCommit-CodePipeline-ECS-Fargate.git
cd AWSWorkshop-20200410-CodeCommit-CodePipeline-ECS-Fargate
aws codecommit create-repository --repository-name workshop-devops-ccef
```
And you will get an output like this:
```
{
    "repositoryMetadata": {
        "repositoryName": "workshop-devops-ccef", 
        "cloneUrlSsh": "ssh://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/workshop-devops-ccef", 
        "lastModifiedDate": 1586480711.24, 
        "repositoryId": "ad8c32e9-ffcc-4976-8573-0c2c6ca2c570", 
        "cloneUrlHttp": "https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/workshop-devops-ccef", 
        "creationDate": 1586480711.24, 
        "Arn": "arn:aws:codecommit:ap-northeast-1:384612698411:workshop-devops-ccef", 
        "accountId": "384612698411"
    }
}
```
Now we use following command to push workshop files into your new CodeCommit repository:
And please replace the <cloneUrlHttp> with your own repo url like "https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/workshop-devops-ccef"
```
git remote remove origin
git remote add origin <cloneUrlHttp>
git push origin master
```
Then we are going to initialize our ECR
```
docker build -t workshop-devops-ccef .

```
Then, create ECR with awscli:
```
aws ecr create-repository --repository-name workshop-devops-ccef
```
And you will see ECR result as:
```
{
    "repository": {
        "registryId": "384612698411", 
        "repositoryName": "workshop-devops-ccef", 
        "repositoryArn": "arn:aws:ecr:ap-northeast-1:384612698411:repository/workshop-devops-ccef", 
        "createdAt": 1586482684.0, 
        "repositoryUri": "384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef"
    }
}
```
And then please replace <repositoryUri> with your value, and <AWS_REGION> with where you are, us-east-1 as default.
```
aws ecr get-login-password --region <AWS_REGION> | docker login --username AWS --password-stdin <repositoryUri>
docker tag workshop-devops-ccef:latest <repositoryUri>:latest
docker push <repositoryUri>:latest

# Example
# aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef
# docker tag workshop-devops-ccef:latest 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef:latest
# docker push 384612698411.dkr.ecr.ap-northeast-1.amazonaws.com/workshop-devops-ccef:latest
```
------
### Step 5:
* Create Security Group, Target Group *2 and Application Loadbalancer *2.
* ***AWS Console > Services > EC2 > Security Group***
* Click ***Create Security Group***
- Name=Workshop_port_80, VPC=Default, Description=Temp Security Group for Workshop
- Inbound Rule > Add Rule > Type=HTTP, Destination=Anywhere
------
* ***AWS Console > Services > EC2 > Load Balancers
* Click ***Create Load Balancer***
- Click "Application Load Balancer"
- Name=***alb-prod***, Scheme=internet-facing, IP Address Type=ipv4
- Load Balancer Protocol=HTTP, Load Balancer Port=80
- AZS > VPC=default, AZ=Select all
- Click Next to Security Group, Select "Workshop_port_80" 
- Click Next to Target Group, Select "New Target Group", Name=tg-prod, Target type=IP, Protocol=HTTP, Port=80
- Click to Review and Create
------
* ***AWS Console > Services > EC2 > Load Balancers***
* Click ***Create Load Balancer***
- Click "Application Load Balancer"
- Name=***alb-beta***, Scheme=internet-facing, IP Address Type=ipv4
- Load Balancer Protocol=HTTP, Load Balancer Port=80
- AZS > VPC=default, AZ=Select all
- Click Next to Security Group, Select "Workshop_port_80" 
- Click Next to Target Group, Select "New Target Group", Name=***tg-beta***, Target type=***IP***, Protocol=HTTP, Port=80
- Click to Review and Create
-------

### Step 6:
* Now we are going to create our ECS Fargate Services for our Beta & Prod Env
* ***AWS Console > Services > ECS***
* Clusters > Create Cluster > Type=Network Only (Fargate) > Cluster Name=fargate-prod
* Clusters > Create Cluster > Type=Network Only (Fargate) > Cluster Name=fargate-beta
* ***In this workshop, we try to simulate how we implement on real corp env, so we separate the services into two fargate clusters.***
* Task Definitions > Create New Task Definition 




### After Workshop -- Clean up
* *** clean up the ECS-Fargate stack with "cdk destroy"***
* *** cloud up the Lambda SAM with "aws cloudformation delete-stack --stack-name=lambda-gin-refarch"***
* *** Terminate the Cloud9 Env from AWS Console > Cloud9 > Environment > Delete***
* *** Remove IAM Role/User from AWS Console > IAM > Role/User > Delete***
