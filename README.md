# Deploying Node.js Application on AWS EC2 Instance

Your company has decided to utilize AWS as the cloud provider for deploying applications, aiming to consolidate platforms and streamline management tasks, including billing. Below are step-by-step exercises to set up and deploy a Node.js application on an EC2 instance using the AWS Command Line Tool.

## Step 1: Create IAM User

**permissions needed for the AWS user**
- Create VPC and Subnet
- Create EC2 instance 
- Create Security Group for EC2

1. Create a new IAM user named "Amina" and assign it to the "devops" group.

     ```bash
     # Check that you have configured AWS admin user locally
     aws configure list
     cat ~/.aws/credentials

     # Create a new IAM user using "your name" with UI and CLI access
     aws iam create-user --user-name Amina
     
     # Create a group "devops"
     aws iam create-group --group-name devops

     # Add your user to "devops" group
     aws iam add-user-to-group --user-name Amina --group-name devops

     # Verify that devops group contains your user
     aws iam get-group --group-name devops

     ```

<img src="https://i.imgur.com/31SIXgg.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

****


2. Ensure the "devops" group has the necessary permissions for the tasks, including login and CLI credentials.


**Give user UI and CLI access**

```sh
# Generate user keys for CLI access & save key.txt file in safe location
aws iam create-access-key --user-name amina > key.txt

# Generate user login credentials for UI & save password in safe location
aws iam create-login-profile --user-name amina --password MyTestPassword123

# Give user permission to change password
aws iam list-policies | grep ChangePassword
aws iam attach-user-policy --user-name amina --policy-arn "arn:aws:iam::aws:policy/IAMUserChangePassword"

```

<img src="https://i.imgur.com/wIPiq7C.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


**Assign permissions to the user through group**

```sh
# Check which policies are available for managing EC2 & VPC services, including subnet and security groups
aws iam list-policies | grep EC2FullAccess
aws iam list-policies | grep VPCFullAccess

# Give devops group needed permissions
aws iam attach-group-policy --group-name devops --policy-arn "arn:aws:iam::aws:policy/AmazonEC2FullAccess"
aws iam attach-group-policy --group-name devops --policy-arn "arn:aws:iam::aws:policy/AmazonVPCFullAccess"

# Check policies for group
aws iam list-attached-group-policies --group-name devops

```

<img src="https://i.imgur.com/703MmZY.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

****


## Step 2: Configure AWS CLI


```sh
# Save your current admin user keys from ~/.aws/credentials in a safe location

# Set credentials for the new user in AWS CLI from key.txt file
$ aws configure
AWS Access Key ID [****]: new-access-key-id
AWS Secret Access Key [****]: new-secret-access-key
Default region name [us-west-1]: new-region
Default output format [None]:

# You can try to validate that ~/.aws/credentials contains the keys of the new user 

```


## Step 3: Create VPC

1. Use the AWS CLI to create a new VPC with one subnet.

     ```sh
     # Create VPC and return VPC id
     aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text

     # Create subnet in the VPC
     aws ec2 create-subnet --vpc-id YOUR_VPC_ID --cidr-block 10.0.1.0/24

     # Return subnet ID
     aws ec2 describe-subnets --filters "Name=vpc-id,Values=YOUR_VPC_ID"

     ```

<img src="https://i.imgur.com/kxT1vU5.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

****

2. Make our subnet public by attaching it internet gateway**

    ```sh
    # Create internet gateway & return the gateway id
    aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text

    # Attach internet gateway to our VPC
    aws ec2 attach-internet-gateway --vpc-id YOUR_VPC_ID --internet-gateway-id YOUR_IGW_ID

    # Create a custom Route table for our VPC & return route table ID
    aws ec2 create-route-table --vpc-id YOUR_VPC_ID --query RouteTable.RouteTableId --output text

    # Create Route rule for handling all traffic between internet & VPC
    aws ec2 create-route --route-table-id YOUR_RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id YOUR_IGW_ID

    # Validate your custom route table has correct configuration, 1 local and 1 internet gateway routes
    aws ec2 describe-route-tables --route-table-id YOUR_RTB_ID

    # Associate subnet with the route table to allow internet traffic in the subnet as well
    aws ec2 associate-route-table --subnet-id YOUR_SUBNET_ID --route-table-id YOUR_RTB_ID

    ```
    
<img src="https://i.imgur.com/hVZlmQ4.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>
   
3. Create a security group within the VPC to allow SSH access on port 22 and browser access to the Node.js application.

     ```sh
     aws ec2 create-security-group --group-name my-sg --description "My Security Group" --vpc-id <vpc-id>
     aws ec2 authorize-security-group-ingress --group-name my-sg --protocol tcp --port 22 --cidr 0.0.0.0/0
     aws ec2 authorize-security-group-ingress --group-name my-sg --protocol tcp --port 80 --cidr 0.0.0.0/0
     ```

<img src="https://i.imgur.com/2ARQ92p.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## Step 4: Create EC2 Instance

1. After creating the VPC, use the AWS CLI to launch an EC2 instance in that VPC.

     ```sh
     # Create key pair, save it locally in pem file and set stricter permissions on it for later use
     aws ec2 create-key-pair --key-name WebServerKeyPair --query "KeyMaterial" --output text > WebServerKeyPair.pem
     chmod 400 WebServerKeyPair.pem

     # Create 1 EC2 instance with the above key, in our subnet and using the security group we created. Check the UI to get an up to date image-id
     aws ec2 run-instances --image-id image-id --count 1 --instance-type t2.micro --key-name WebServerKeyPair --security-group-ids sg-id --subnet-id subnet-id --associate-public-ip-address

     # Validate that EC2 instance is in a running state, and get its public ip address to connect via ssh
     aws ec2 describe-instances --instance-id i-0146854b7443af453 --query "Reservations[*].Instances[*].{State:State.Name,Address:PublicIpAddress}"

     ```
<img src="https://i.imgur.com/rrxFRL0.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


<img src="https://i.imgur.com/Itrxbd9.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## Step 5: SSH into the Server and Install Docker

1. Once the EC2 instance is successfully launched, SSH into the server.
   
   ```sh

   # ssh into EC2 instance using the public IP address we got earlier
   # Note: EC2 must be in a running state & relative path to WebServerKeyPair.pem must be correct
   ssh -i "WebServerKeyPair.pem" ec2-user@public-ip-address

   # Install Docker, start docker service and allow ec2-user to run docker commands without sudo by adding it to docker group
   sudo yum update -y && sudo yum install -y docker
   sudo systemctl start docker
   sudo usermod -aG docker ec2-user

   ```

<img src="https://i.imgur.com/IovAb9f.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

****

# Continuous Deployment Setup Guide


## Step 6: Add docker-compose for deployment

1. **Add docker-compose to your NodeJS application:**
   - Docker-compose allows you to define and run multi-container Docker applications. By adding a docker-compose.yml file to your project, you can specify the configuration for running your application, including any dependencies like databases or services.
   **docker-compose.yaml:**

```sh
version: '3.8'
services:
    nodejs-app:
      image: ${IMAGE}
      ports:
        - 3000:3000

```

<img src="https://i.imgur.com/kiHP1Za.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## Step 7: Add "deploy to EC2" step to your existing pipeline

1. **Modify the Jenkinsfile:**
   - Update the Jenkinsfile in your project to include a deployment step that deploys your application to an EC2 instance.

     **Create ssh keys credentials in Jenkins**

     - name the keys credentials: `ec2-server-key` and add the private ssh key from `WebServerKeyPair.pem` as its content. You will use this to ssh into the EC2 server from Jenkins

       **Create server-cmds.sh file**

       ```sh
       export IMAGE=$1
       docker-compose -f docker-compose.yaml up --detach
       echo "success"
       ```
       
<img src="https://i.imgur.com/wykE1Ym.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

- Build Automation & CI/CD with Jenkins

**Jenkinsfile:**
```sh
pipeline {
    agent any
    tools {
        nodejs "node"
    }
    stages {
        stage('Increment Version') {
            // Add your commands for incrementing version
        }
        stage('Run Tests') {
            // Add your commands for running tests
        }
        stage('Build and Push Docker Image') {
            // Add your commands for building and pushing Docker image
        }
        stage('Deploy to EC2') {
            steps {
                script {
                    // Define variables
                    def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}"
                    def ec2Instance = "ec2-user@35.176.254.134"

                    // Run SSH commands using sshagent
                    sshagent(credentials: ['ec2-server-key']) {
                        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
                    }     
                }
            }
        }
        stage('Commit Version Update') {
            // Add your commands for committing version update
        }
    }     
}

```

<img src="https://i.imgur.com/Ppsbhao.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## Step 8: Configure access from browser (EC2 Security Group)

1. **Configure EC2 Security Group:**
   - After deploying your application to the EC2 instance, you need to configure the EC2 Security Group to allow incoming traffic from the browser.
     
     **Open application's port 3000 in security group to make app accessible from browser**

```sh
aws ec2 authorize-security-group-ingress --group-id sg-id --protocol tcp --port 3000 --cidr 0.0.0.0/0

```

<img src="https://i.imgur.com/yx6dOF3.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## Step 9: Configure automatic triggering of multi-branch pipeline

1. **Implement branch-based logic:**
   - Modify your Jenkinsfile to include branch-based logic that controls when builds and deployments occur. For example, you may want to deploy changes only from the master branch.
  
   - **Add branch based logic to Jenkinsfile**

```sh
# when the current branch is master, execute all steps. If it's not master, execute only the "run tests" step
pipeline {
    agent any
      tools {
        nodejs "node"
      }
      stages {
        stage('increment version') {
          when {
            expression {
              return env.GIT_BRANCH == "master"
            }
          }
          steps {
            script {
                ...  
            }
          }
        }
        stage('Run tests') {
          steps {
            script {
                ...  
            }
          }
        }
        stage('Build and Push docker image') {
          when {
            expression {
              return env.GIT_BRANCH == "master"
            }
          }
          steps {
            script {
                ...  
            }
          }
        }
        stage('deploy to EC2') {
          when {
            expression {
              return env.GIT_BRANCH == "master"
            }
          }
          steps {
            script {
                ...  
            }
          }
        }
        stage('commit version update') {
          when {
            expression {
              return env.GIT_BRANCH == "master"
            }
          }
          steps {
            script {
                ...  
            }
          }  
        }
    }     
}

```


2. **Set up webhook in GitHub:**
   - Configure a webhook in your GitHub repository to automatically trigger the Jenkins pipeline whenever changes are pushed to the repository.
  
