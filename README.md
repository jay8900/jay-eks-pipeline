# Amazon EKS Cluster Creation

This repository contains the following files:

- [eks.yml](eks.yml): a CloudFormation template that defines an EKS cluster, including a VPC, the EKS control plane (master nodes) and the EKS worker nodes.
- [up.sh](up.sh): a Bash script that applies the CloudFormation template to your AWS account and finalises the cluster creation, including `kubectl` configuration.

## Usage

### Prerequisites

- Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html):
    ~~~bash
    pip install awscli
    ~~~
- Install the [AWS IAM Authenticator](https://docs.aws.amazon.com/eks/latest/userguide/configure-kubectl.html):
    ~~~bash
    go get -u -v github.com/kubernetes-sigs/aws-iam-authenticator/cmd/aws-iam-authenticator
    ~~~
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/):
   ~~~bash
   brew install kubernetes-cli
   ~~~

### Run

1. Edit parameters in `up.sh`:
    - `NUM_WORKER_NODES`: number of worker nodes for your EKS cluster
    - `WORKER_NODES_INSTANCE_TYPE`: EC2 instance type for the worker nodes
    - `STACK_NAME`: name of the CloudFormation stack for your EKS cluster
    - `KEY_PAIR_NAME`: name of an existing EC2 key pair for connecting to the worker nodes with SSH

2. Run:
    ~~~bash
    ./iac.sh
    ~~~

#### Note

Any arguments that you pass to `up.sh` will be forwarded to the AWS CLI commands within the script. Thus, it is possible to specify an explicit region fo the cluster as follows:

~~~bash
./iac.sh --region eu-west-1
~~~


We will write a buildspec.yml which will eventually build a docker image, push the same to ECR Repository and Deploy the updated k8s Deployment manifest to EKS Cluster.
To achive all this we need also create or update few roles:

STS Assume Role: EksCodeBuildKubectlRole
Inline Policy: eksdescribe
CodeBuild Role: codebuild-eks-devops-cb-for-pipe-service-role
ECR Full Access Policy: AmazonEC2ContainerRegistryFullAccess
STS Assume Policy: eks-codebuild-sts-assume-role
STS Assume Role: EksCodeBuildKubectlRole

Create ECR Repository for our Application Docker Images:

Go to Services -> Elastic Container Registry -> Create Repository
Name: eks-devops-nginx
Tag Immutability: Enable
Scan On Push: Enable
Click on Create Repository
Make a note of Repository name


Create CodeCommit Repository
Step-05: Create CodeCommit Repository

Code Commit Introduction

Create Code Commit Repository with name as eks-devops-nginx
Create git credentials from IAM Service and make a note of those credentials.
Clone the git repository from Code Commit to local repository, during the process provide your git credentials generated to login to git repo

Copy all files from github to local repository
buildspec.yml
Dockerfile
app1
index.html
kube-manifests
01-DEVOPS-Nginx-Deployment.yml
02-DEVOPS-Nginx-NodePortService.yml
03-DEVOPS-Nginx-ALB-IngressService.yml
Commit code and Push to CodeCommit Repo
Verify the same on CodeCommit Repository in AWS Management console.

Application Manifests Overview:

buildspec.yml
Dockerfile
app1
index.html
kube-manifests
01-DEVOPS-Nginx-Deployment.yml
02-DEVOPS-Nginx-NodePortService.yml
03-DEVOPS-Nginx-ALB-IngressService.yml

Create STS Assume IAM Role for CodeBuild to interact with AWS EKS:

# Export your Account ID
export ACCOUNT_ID=180789647333

# Set Trust Policy
TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

# Verify inside Trust policy, your account id got replacd
echo $TRUST

# Create IAM Role for CodeBuild to Interact with EKS
aws iam create-role --role-name EksCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

# Define Inline Policy with eks Describe permission in a file iam-eks-describe-policy
echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-eks-describe-policy

# Associate Inline Policy to our newly created IAM Role
aws iam put-role-policy --role-name EksCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-eks-describe-policy

# Verify the same on Management Console


Update EKS Cluster aws-auth ConfigMap with new role created in previous step:

# Verify what is present in aws-auth configmap before change
kubectl get configmap aws-auth -o yaml -n kube-system

# Export your Account ID
export ACCOUNT_ID=180789647333

# Set ROLE value
ROLE="    - rolearn: arn:aws:iam::$ACCOUNT_ID:role/EksCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

# Get current aws-auth configMap data and attach new role info to it
kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

# Patch the aws-auth configmap with new role
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

# Verify what is updated in aws-auth configmap after change
kubectl get configmap aws-auth -o yaml -n kube-system

 Review the buildspec.yml for CodeBuild & Environment Variables:
 
REPOSITORY_URI = $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/eks-devops-nginx
EKS_KUBECTL_ROLE_ARN = arn:aws:iam::180789647333:role/EksCodeBuildKubectlRole
EKS_CLUSTER_NAME = jay-cluster



Create CodePipeline:

Create CodePipeline
Go to Services -> CodePipeline -> Create Pipeline
Pipeline Settings
Pipeline Name: eks-devops-pipe
Service Role: New Service Role (leave to defaults)
Role Name: Auto-populated
Rest all leave to defaults and click Next
Source
Source Provider: AWS CodeCommit
Repository Name: eks-devops-nginx
Branch Name: master
Change Detection Options: CloudWatch Events (leave to defaults)
Build
Build Provider: AWS CodeBuild
Region: US East (N.Virginia)
Project Name: Click on Create Project
Create Build Project
Project Configuration
Project Name: eks-devops-cb-for-pipe
Description: CodeBuild Project for EKS DevOps Pipeline
Environment
Environment Image: Managed Image
Operating System: Amazon Linux 2
Runtimes: Standard
Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0
Image Version: Always use the latest version for this runtime
Environment Type: Linux
Privileged: Enable
Role Name: Auto-populated
Additional Configurations
All leave to defaults except Environment Variables
Add Environment Variables
REPOSITORY_URI = ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/eks-devops-nginx
EKS_KUBECTL_ROLE_ARN = arn:aws:iam::ACCOUNT_ID:role/EksCodeBuildKubectlRole
EKS_CLUSTER_NAME =  jay-cluster
Buildspec
leave to defaults
Logs
Group Name: eks-deveops-cb-pipe
Stream Name:
Click on Continue to CodePipeline
We should see a message Successfully created eks-devops-cb-for-pipe in CodeBuild.
Click Next
Deploy
Click on Skip Deploy Stage
Review
Review and click on Create Pipeline


Updae CodeBuild Role to have access to ECR full access:

First pipeline run will fail as CodeBuild not able to upload or push newly created Docker Image to ECR Repostory
Update the CodeBuild Role to have access to ECR to upload images built by codeBuild.
Role Name: codebuild-eks-devops-cb-for-pipe-service-role
Policy Name: AmazonEC2ContainerRegistryFullAccess
Make changes to index.html (Update as V2), locally and push change to CodeCommit


Update CodeBuild Role to have access to STS Assume Role we have created using STS Assume Role Policy:

Go to Services IAM -> Policies -> Create Policy
In Visual Editor Tab
Service: STS
Actions: Under Write - Select AssumeRole
Resources: Specific
Add ARN
Specify ARN for Role: arn:aws:iam::180789647333:role/EksCodeBuildKubectlRole
Click Add

# For Role ARN, replace your account id here, refer step-07 environment variable EKS_KUBECTL_ROLE_ARN for more details
arn:aws:iam::<your-account-id>:role/EksCodeBuildKubectlRole
Click on Review Policy
Name: eks-codebuild-sts-assume-role
Description: CodeBuild to interact with EKS cluster to perform changes
Click on Create Policy

 Associate Policy to CodeBuild Role:

Role Name: codebuild-eks-devops-cb-for-pipe-service-role
Policy to be associated: eks-codebuild-sts-assume-role
