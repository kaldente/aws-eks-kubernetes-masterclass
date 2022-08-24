# Create EKS Cluster & Node Groups

## Step-00: Introduction
- Understand about EKS Core Objects
  - Control Plane
  - Worker Nodes & Node Groups
  - Fargate Profiles
  - VPC
- Create EKS Cluster
- Associate EKS Cluster to IAM OIDC Provider
- Create EKS Node Groups
- Verify Cluster, Node Groups, EC2 Instances, IAM Policies and Node Groups


## Step-01: Create EKS Cluster using eksctl
- It will take 15 to 20 minutes to create the Cluster Control Plane 
```
# Create Cluster
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 
                      
########################################################################################
jkyung@jkyung-J93MVWN345 ~ % eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
########################################################################################
                      


# Get List of clusters
eksctl get cluster                  
```


## Step-02: Create & Associate IAM OIDC Provider for our EKS Cluster
- To enable and use AWS IAM roles for Kubernetes service accounts on our EKS cluster, we must create &  associate OIDC identity provider.
- To do so using `eksctl` we can use the  below command. 
- Use latest eksctl version (as on today the latest version is `0.21.0`)
```                   
# Template
eksctl utils associate-iam-oidc-provider \
    --region region-code \
    --cluster <cluter-name> \
    --approve

# Replace with region & cluster name
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve
```

################################################################################################
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster=eksdemo1 --approve
################################################################################################

## Step-03: Create EC2 Keypair
- Create a new EC2 Keypair with name as `kube-demo`
- This keypair we will use it when creating the EKS NodeGroup.
- This will help us to login to the EKS Worker Nodes using Terminal.

################################################################################################
Google Drive/My Drive/Tech/AWS/AWS_EKS_Masterclass/kube-demo.pem
################################################################################################

## Step-04: Create Node Group with additional Add-Ons in Public Subnets
- These add-ons will create the respective IAM policies for us automatically within our Node Group role.
 ```
# Create Public Node Group   
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-public1 \
                        --node-type=t3.medium \
                        --nodes=2 \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access 
```

###########################################################################################################
jkyung@jkyung-J93MVWN345 aws_eks_masterclass % eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-public1 \
                        --node-type=t2.micro \
                        --nodes=2 \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access
###########################################################################################################

## Step-05: Verify Cluster & Nodes

### Verify NodeGroup subnets to confirm EC2 Instances are in Public Subnet
- Verify the node group subnet to ensure it created in public subnets
  - Go to Services -> EKS -> eksdemo -> eksdemo1-ng1-public
  - Click on Associated subnet in **Details** tab
  - Click on **Route Table** Tab.
  - We should see that internet route via Internet Gateway (0.0.0.0/0 -> igw-xxxxxxxx)

### Verify Cluster, NodeGroup in EKS Management Console
- Go to Services -> Elastic Kubernetes Service -> eksdemo1

### List Worker Nodes
```
# List EKS clusters
eksctl get cluster

# List NodeGroups in a cluster
eksctl get nodegroup --cluster=<clusterName>

########################################################################################################################################################
jkyung@jkyung-J93MVWN345 aws_eks_masterclass % eksctl get nodegroups --cluster eksdemo1
CLUSTER         NODEGROUP               STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID        ASG NAME                                                        TYPE
eksdemo1        eksdemo1-ng-public1     ACTIVE  2022-08-23T16:51:02Z    2               4               2                       t2.micro        AL2_x86_64      eks-eksdemo1-ng-public1-4ec165ce-a882-60b4-48db-453456bdb50f    managed
########################################################################################################################################################


# List Nodes in current kubernetes cluster
kubectl get nodes -o wide

############################################################################################################################################
After installing kubctl `brew install kubectl`, I need to run `aws eks update-kubeconfig --name <cluster_name>` in order to connect to a EKS 
cluster to view nodes.

jkyung@jkyung-J93MVWN345 ~ % kubectl cluster-info

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
The connection to the server localhost:8080 was refused - did you specify the right host or port?
jkyung@jkyung-J93MVWN345 ~ % aws eks update-kubeconfig --name kubedemo1

An error occurred (ResourceNotFoundException) when calling the DescribeCluster operation: No cluster found for name: kubedemo1.
jkyung@jkyung-J93MVWN345 ~ % eksctl get cluster
NAME		REGION		EKSCTL CREATED
eksdemo1	us-east-1	True
jkyung@jkyung-J93MVWN345 ~ % aws eks update-kubeconfig --name eksdemo1
Added new context arn:aws:eks:us-east-1:728417853576:cluster/eksdemo1 to /Users/jkyung/.kube/config
jkyung@jkyung-J93MVWN345 ~ % kubectl get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-17-183.ec2.internal   Ready    <none>   24h   v1.23.9-eks-ba74326
ip-192-168-39-183.ec2.internal   Ready    <none>   24h   v1.23.9-eks-ba74326
###############################################################################################################################################
# Our kubectl context should be automatically changed to new cluster
kubectl config view --minify
```

### Verify Worker Node IAM Role and list of Policies
- Go to Services -> EC2 -> Worker Nodes
- Click on **IAM Role associated to EC2 Worker Nodes**

### Verify Security Group Associated to Worker Nodes
- Go to Services -> EC2 -> Worker Nodes
- Click on **Security Group** associated to EC2 Instance which contains `remote` in the name.

### Verify CloudFormation Stacks
- Verify Control Plane Stack & Events
- Verify NodeGroup Stack & Events

########################################################################################################################################################
NAT gateway was created as part of EKS cluster creation (go Cloudformation > Resources).
NAT Gateway is a highly available AWS managed service that makes it easy to connect to the Internet from instances within a private subnet in an Amazon Virtual Private Cloud (Amazon VPC)
########################################################################################################################################################

### Login to Worker Node using Keypai kube-demo
- Login to worker node
```
# For MAC or Linux or Windows10
ssh -i kube-demo.pem ec2-user@<Public-IP-of-Worker-Node>

# For Windows 7
Use putty
```

## Step-06: Update Worker Nodes Security Group to allow all traffic
- We need to allow `All Traffic` on worker node security group

## Additional References
- https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
- https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html
