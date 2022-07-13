# Implementattion of DevOps CI/CD pipeline for Website using Microservive Architecture.

This project contains the following objectives:

- Source code Management using Git and Github.
- Building the Project using the Maven.
- Building the Pipeline using Jenkins.
- Deploying the code on the tomcat server, using the Microservice tools like Docker and Kubernetes.
- Maintaining the CI/CD pipeline throughout the Process.

## Project Architecture

![Architeture Image](https://github.com/AbhixGupta/CI-CD_Ecommerce_Website/tree/master/architecture.png?raw=true)

## Installation

We need to maintain the three servers in the AWS cloud. Each server is maintained in the EC2 instances, for this project we are using the "t2.micro" instances.

### AWS Account Creation

For aws Account creation you can find the installation documention from here.

### Jenikns Server

Second step is to setup the jekins server, you can have lauch one EC2 instance with custom TCP rules and opeen the the "port:8080" with CIDR from anywhere. Run the follwing command to install the jenkins.

```bash
  sudo su -
  sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
  sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
  amazon-linux-extras install epel
  amazon-linux-extras install java-openjdk11
  yum install jenkins -y
  yum install git -y
```

Now run the following command to make the jenkins to start whenever the system starts.

```bash
  systemctl start jenkins
```

### Ansible Server

Ansible is the congiguration Management Tool. Here we will use it for providing the configuration for the Docker and the Kubernetes conatiner and cluster respectively.
First Launch the EC2 instance same as lauched for the jenkins server. Opent he port:8080.

Now login as root user and add one user named as "ansibleadmin" and the password for this user.

```bash
  sudo su -
  useradd ansibleadmin
  passwd ansibleadmin // enter the password for this user
```

Add this user to the sudoers file and enable the password Authentication.

```bash
  visudo
  ansibleadmin  ALL=(ALL)   NOPASSWD: ALL //Add this command next to the #wheel comment
```

Now edit the host file,

```bash
  vi /etc/ansible/hosts
  PRIVATE IP OF ANSIBLE HOST
```

Save and exit. Now run the following command:

```bash
  ansible all -a uptime
  ssh-copy-id <PrivateIpAnsible>
  ssh-copy-id localhost
  ansible all -a uptime
  cd /opt/docker
  vi regapp.yml
  ansibl-playbook regapp.yml --check
  docker images
  ansible-playbook regapp.yml
```

Before proceeding furthur create the Docker Hub account. After creating the account login in the ansible server. In ansible server login as "ansibleadmin".

```bash
  cd /opt/docker
  docker login
  docker push <ImageID> <DockerhubUsername>/reagap:latest
  docker push <DockerhubUsername>/reagapplatest
  ansible-playbook regapp.yml --limit <ansiblePrivateIp>
  vi deploy-regapp.yml
  ansible-playbook deploy-regapp.yml --check
```

### Kubernetes Server

Goto AWS console, Launch one EC2 instance same as you have lauched for other servers. Name the instance as "EKS-server" and select the DevOps Security Group.

- First you have to install the latest AWS CLI version and setup the Kubectl, for this project we have to download the Kubernetes 1.21 version.

```bash
  sudo su -
  aws --version
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install
  aws --version
  curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.21.2/2021-07-05/bin/darwin/amd64/kubectl
  chmod +x ./kubectl
  mv ./kubectl /usr/local/bin
  kubectl version --short --client
```

- Next you have to setup the eksctl

```bash
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
```

- Create IAM role and attach this role with this Kubernets Server. Goto IAM ROLE console in AWS, create one role and attach these poilicies with it. After attaching the policies click on create.

```bash
  IAM user should have access to
  IAM
  EC2
  CloudFormation
```

- Now after attaching the policies we have to make the cluster, for this run the following commands:

```bash
  eksctl create cluster --name abhigeu  \
  --region ap-south-1 \
  --node-type t2.micro
```

- Below are some basic commands of the kubernetes which we can use to get the information about the pods, services and the containers.

```bash
  kubectl get all
  kubectl get nodes
  kubectl get po
  kubectl get deployment

  vi regapp-deployment.yml
  vi regapp-service.yml
```

### Integrate the Kubernets with Ansible

- Login to the Kubernets server:

```bash
  sudo su -
  useradd ansibleadmin
  visudo
  **** Under the root add****
  ansibleadmin  ALL=(ALL) NOPASSWD: ALL     //Save and exit
```

Open the ssh configuration file and enable the Password Authentication=Yes, and comment out where you will find the "PasswordAuthentication=NO".

```bash
  vi /etc/ssh/sshd_config
  service sshd reload
  passwd ansibleadmin  // setup the password for this user

```

Save and exit the file.

- Login to the Ansible server as "ansibleadmin":

```bash
  sudo su - ansibleadmin
  cd /opt/docker
  mv regapp.yml create_image_regapp.yml
  mv deploy_regapp.yml docker_deployment.yml
```

Open the hosts file and add the following host:

```bash
  localhost
  [Kubernetes]
  PrivateIpKubernetes

  [ansible]
  PrivateIpAnsible
```

Save and exit the file.

Now,

```bash
  ssh-copy-id PublicIpKubernetes
  ansible -i hosts all -a uptime
  vi kube_deployment.yml
  vi kube_service.yml
  ansible-playbook -i /opt/docker/hosts/kube-deployment.yml
  ssh-copy-id root@KubernetesIP
  ansible-playbook -i /opt/docker/hosts/kube-service.yml

```

### The last Task is to create Two Jenkins Jobs one if Continuos Integration (CI) and other one is for Continuos Deployment (CD):

#### Plugins to Install

```bash
  Deploy to container
  Publish over ssh
  Git
  Github
  Maven
```

- For CI Job

1. Name the Project and select the Maven Project.
2. In Git section paste the Git source code URL.
3. In Post Build Action select the "send build artifacts over ssh".

-- Name:

```bash
  ansible-server
```

-- Source file:

```bash
  webapp/target/*.war
```

-- Remove prefix:

```bash
  webapp/target
```

-- Remote Directory:

```bash
  //opt//docker
```

-- Exec command:

```bash
ansible-playbook /opt/docker/create-image-regapp.yml
```

- For CI Job

1. Name the Project and select the Freestyle Project.

-- Name:

```bash
  ansible-server
```

-- Exec command:

```bash
  ansible-playbook -i /opt/docker/kube_deploy.yml
  ansible-playbook -i /opt/docker/kube_service.yml
```

4. In Poll SCM section select write "\* \* \* \* \*".

Apply and Save
