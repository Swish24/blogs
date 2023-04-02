---
title: Deploying an EC2 Nginx Web Server on AWS using Terraform
subtitle: Deploying an EC2 Nginx Web Server on AWS using Terraform
slug: deploy-ec2-nginx-docker-webserver-using-terraform
tags: aws, terraform, devops, docker
cover: https://github.com/Swish24/Terraform-EC2/raw/main/images/TerraformEC2Banner.png
domain: blog.dvsn.ai
---

Terraform is an open-source infrastructure as code tool that allows you to automate the provisioning and configuration of infrastructure resources on AWS, and other Cloud providers. It enables you to define your infrastructure as code and easily manage, version and share your infrastructure configurations on AWS.
Today we are going to focus on EC2 instances, Security groups, and Application load balancers.

Please note:
<br>
This article assumes you have configured AWS CLI credentials on your local machine.

If you are looking to dive right in with the completed project files, they are available <a href="https://github.com/Swish24/Terraform-EC2">here on github</a>
<br>
<br>
First, we are going to create a new file called <a href="https://github.com/Swish24/Terraform-EC2/blob/main/Providers.tf" target="_blank">Providers.tf</a>
<br>
Within here we are going to configure the <a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs" target="_blank"> AWS Terraform provider</a>, this will allow us to interact with the AWS API's

```terraform
provider "aws" {
  //Location of the AWS Credentials config file. This contains information on how different profiles or roles can connect.
  shared_config_files      = ["~/.aws/config"]

  //Contains all AWS_ACCESS_KEY,AWS_SECRET_ACCESS_KEY,AWS_SESSION_TOKEN configurations that are within your enviornment.
  shared_credentials_files = ["~/.aws/credentials"]

  //Name of the profile we have configured within the shared config file, or while using aws configure --profile <profile name>
  profile                  = "terraform-dev"

  //Default region where we would like to deploy these resources
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "{$var.defaultTagName}"
      Owner       = "{$var.defaultTagOwner}"
    }
  }
}
```

We are also going to configure our default_tags, we will reference this in a later mentioned variables file. 
<br>

<h2><a href="https://github.com/Swish24/Terraform-EC2/blob/main/Variables.tf" target="_blank">Variables.tf</a></h2>
Here is our variables file, within here we will store the values that we are going to be using.
<br>
This helps store things like domain name, API names, and more in a single place, while referencing it later within code.
<br>
Now here we are only defining a few variables to keep it simple.

```terraform
variable "defaultTagName" {
  type = string
  default = "Development"
  description = "The default supplied development envionrment name"
}

variable "defaultTagOwner" {
    type = string
    default = "DJ"
    description = "the default supplied deveopment owner name"
}

variable "instance_name" {
    type = string
    default = "EC2Demo"
    description = "Name of our EC2 instance that we are going to confiugre"
}
```

Let's start with configuring our security groups


<h2><a href="https://github.com/Swish24/Terraform-EC2/blob/main/SecurityGroups.tf" target="_blank">SecurityGroups.tf</a></h2>

Security groups define how we allow traffic in and out of our AWS resources, at the resource level.
<br>
Create a new file named <a href="https://github.com/Swish24/Terraform-EC2/blob/main/SecurityGroups.tf" target="_blank">SecurityGroups.tf</a></h2>
<br>
This will contain the configuration of all of our security groups
<br>
We are going to be creating our EC2 instance within the default public VPC. 
<br>
<br>
For more complex enterprise enviornments, you will most likely be building within a private VPC.
There are many additional layers behind the scenes such as NACL's, VPC's, Internet Gateway's and more when within a private VPC.
<br>
Security groups can be attached to many different AWS services, for now, we are going to be focusing on EC2 Instances & Load Balancers.

Within the security groups file we will configure all of our desired security groups & rules to easily locate them.
<br>
<i>Due to the length of this policy, you'll want to grab it from github here: <a href="https://github.com/Swish24/Terraform-EC2/blob/main/SecurityGroups.tf" target="_blank">SecurityGroups.tf</a></i>

Here is a snippet of the policiy,

```terraform
//As this is a lab enviornment, let's grab our public home IP, and populate it within the security group.
//This way, we do not have port 22 open to the world wide web
data "http" "ip" {
  url = "https://ifconfig.me/ip"
}

data "aws_vpc" "default" {
  default = true
}

resource "aws_security_group" "Safe_Secure_Inbound" {
  name        = "Safe Secure Inbound"
  description = "Security group for inbound and outbound traffic"
  vpc_id      = data.aws_vpc.default.id
  
}
```

<h2><a href="https://github.com/Swish24/Terraform-EC2/blob/main/EC2.tf" target="_blank">EC2.tf</a></h2>

We are going to define our data source to select our desired AMI for our EC2 Instance
<br>
<i>Due to the length of this file, please grab the full code here: <a href="https://github.com/Swish24/Terraform-EC2/blob/main/EC2.tf" target="_blank">EC2.tf</a></i>

```terraform
//Define the AMI that we will use in our instance configuration
data "aws_ami" "amz-linux-ami" {
    most_recent = true
    owners = ["amazon"]
    filter {
        name ="name"
        values = [
            "*l2023-ami-2023.0.2*"
        ]
    }
}
```
Here we are searching for the latest Amazon Linux 2023 image.
<br>
Notice the wildcards being used here, you have a lot of control to filter for specific values.
<br>
This is helpful because maybe you want to use an AMI updated to a specific date.

To get the list of AMI's, you can use this CLI command <a href="https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-images.html" target="_blank">AWS CLI</a>

```terraform
aws ec2 describe-images --owners amazon --region us-west-1 --filters "Name=name,Values=*l2023-ami-2023.0.*" --query 'Images[*].[Name]' --output text
```

Now, we are going to create the resource object to create our EC2 Instance.

```terraform
resource "aws_instance" "ec2_instance" {
    //Select the EC2 instance size
    instance_type = "t2.micro"

    //Here we will define our AMI variable as we configured previously
    ami = data.aws_ami.amz-linux-ami.id

    //Here we will configure our security group rules used for the instance
    vpc_security_group_ids = ["${aws_security_group.Safe_Secure_Inbound.id}","${aws_security_group.AWS_Session_Connect.id}",
    "${aws_security_group.Outbound_Connections.id}"]

    //Name of our SSH key pair
    key_name ="AWSInstanceKey"

    //Define the root EBS volume parameters
    root_block_device {

    //EBS volume size for our root device
    volume_size = 30 # in GB
    
    //Configure GP3 as the volume type
    volume_type = "gp3"

    //Encryt the volume using KMS. We are going to select no, however this will be covered in a later article
    encrypted   = false

    //Delete the volume when the instance is deleted
    delete_on_termination = true
  }

  //Because we defined our global default name and owner tags, we are not going to configure those here
  //Here we are going to set our "Name" to our variable of instance_name
    tags = {
        Name = "${var.instance_name}"
    }

    user_data = "${file("ec2-userdata.sh")}"

    //Here we have a custom configured user_data configuration to pass to the instance.
    //The instance will execute this script upon booting into the OS, and perform these commands
    //Here we are running all latest updates from the configured distributions. We are then going to install docker & docker compose.
    //Then we are going to configure an nginx linuxserver.io container
    user_data_replace_on_change = true

    //Here we are going to add a dependency on this instance, to ensure the security groups we want attached are already configured

    depends_on = [
      aws_security_group.Safe_Secure_Inbound,
      aws_security_group.AWS_Session_Connect,
      aws_security_group.Outbound_Connections
    ]
}
```
Next, we are going to create the "ec2-userdata.sh" file that we referenced above.
<br>
This will contain a shell script to execute during our instance launch
<br>
```bash
#!/bin/bash

# Perform updates to the instances
dnf update -y
# Install docker
dnf install docker.x86_64 -y
# Start the docker service
service docker start
# Add the EC2 User as administrator so we do not need to run sudo
usermod -a -G docker ec2-user
# Enable docker to run as a service 
systemctl enable docker.service
systemctl enable containerd.service
# Run docker command to install nginx as our webserver
# https://docs.linuxserver.io/images/docker-nginx
docker run -d \
--name=nginx \
-e PUID=1000 \
-e PGID=1000 \
-e TZ=America/Los_Angeles \
-p 80:80 \
-p 443:443 \
-v /mnt/nginx/config:/config \
--restart unless-stopped \
lscr.io/linuxserver/nginx:latest
```

<h2>LoadBalancer.tf</h2>
Last but not least, we have our load balancers. Within this file we will create our Application load balancer, load balancer listeners, and instance target group.
<br>
Create a new file named LoadBalancer.tf
<br>
<i>Due to the length of this file, please grab the full code here: <a href="https://github.com/Swish24/Terraform-EC2/blob/main/LoadBalancer.tf" target="_blank">LoadBalancer.tf</a></i>

<br>

```terraform
//Define our load balancer
resource "aws_lb" "load_balancer" {
  //Put it in all subnets received by above
  subnets         = data.aws_subnets.all.ids

  //Name of the load balancer
  name               = "EC2Demo"
  //This is an internet facing load balancer, so we will select false
  internal           = false

  //Define our load balancer type, there are three, Application, Network and Gateway
  //As we are running a web server, we will choose application
  load_balancer_type = "application"

  //Define the security groups attached to our load balancer.
  //We are going to attach the load balancer SG we created earlier.

  security_groups    = ["${aws_security_group.Loadbalancer_SG.id}"]
  #subnets            = [for subnet in aws_subnet.public : subnet.id]

  //Deletion protection ensures this requires an extra step before deleting it
  enable_deletion_protection = false
}
```

At this point, we have all the files together, you should have all these files created within your project
![Terraform file structure](https://github.com/Swish24/Terraform-EC2/raw/main/images/filestructure.png)
<br> 
<br>
Let's run a terraform init, to initalize our terraform deployment, you should receive this output

![Terraform Init](https://github.com/Swish24/Terraform-EC2/raw/main/images/terraforminit.png) 

Great, now our terraform enviornment is initialized and ready for use!

Let's run a terraform plan to have terraform briefly check and confirm our configuration

```terraform
terraform plan

```
![Terraform Plan](https://github.com/Swish24/Terraform-EC2/raw/main/images/terraformplan.png) 

This will display all of the changes that will be made to the enviornment. As your deployment grows, you will be changing and destroying objects. 
Keep close aware to these indicators unless you are intending to delete specific resources.

To finalize and accept this, let's execute

```terraform
terraform apply
```
Terraform will now display the same results as the plan, and require you to confirm the changes by entering "yes".

![Terraform Plan](https://github.com/Swish24/Terraform-EC2/raw/main/images/terraformapply.png) 

Great, our EC2 instance, four security groups , load balancer, and target groups have been created with terraform!

![Terraform Successfully Delpoyed](https://github.com/Swish24/Terraform-EC2/raw/main/images/terraformcomplete.png) 


Let's head to our instance within the console, confirm the successul deployment, and that our user data executed successfully.

<a href="https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Instances:instanceState=running" target="_blank">EC2 Console <a>
Here within the console we can see our instance has successfully deployed and is online

![EC2 Instance Deployed](https://github.com/Swish24/Terraform-EC2/raw/main/images/ec2instancedeployed.png) 

Let's connect to the instance via the AWS console EC2 Instance connect, this will allow us console access to our instance.
Select your instance and click connect

![Connect to EC2 Instance](https://github.com/Swish24/Terraform-EC2/raw/main/images/ec2instanceconnect.png) 

Ensure you select "EC2 Instance Connect" & Select connect.
The ec2-user is the default used user with Amazon Linux AMI's.

![EC2 Instance Connect](https://github.com/Swish24/Terraform-EC2/raw/main/images/ec2instanceconnect2.png) 

Now, we are connected to our instance 
Let's enter "docker ps -a" and hit enter.
![EC2 Docker Running](https://github.com/Swish24/Terraform-EC2/raw/main/images/ec2dockerrunning.png)

This will output our running docker containers, and we can see the nginx container we deployed within our EC2 user-data config was successful.

Let's check our webserver within our browser.
<br>
Grab your EC2_instance_public_dns that was output during our terraform apply, and paste the url within your browser.
<br>
![EC2 Webserver Offline](https://github.com/Swish24/Terraform-EC2/raw/main/images/ec2InstanceOffline.png) 

As expected, we are not able to connect to the ec2_instance_public_dns. This is by design as we do not want any connections directly to our instance.
<br>
We want all inbound traffic from the internet, to route through our application load balancer.
<br>
Now grab the "lb_public_dns" that was output during our terraform apply.

As expected, we can now access our website, as we are connecting through the load balancer

![Successful EC2 Webserver](https://github.com/Swish24/Terraform-EC2/raw/main/images/ec2InstanceOnline.png) 


Here we have succesfully deployed an EC2 instance, with a load balancer in front to handle traffic.
<br>
We ensured we are only allowing SSH access to our instance from secure, trusted IP Addresses to prevent malicious activity.
<br>

Stay tuned for the next article where we will configure a custom domain name & HTTPS certificate for our webserver.
This will allow us to reach our website with our own domain and support secure, encrypted traffic to our instance.

Keep labbing!