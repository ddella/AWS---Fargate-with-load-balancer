
# Amazon ECS and load balancer with Docker containers

In this workshop, you'll build a Docker image with Nginx web server and PHP 8 and deploy it on Amazon ECS with a load balancer. If you already have your own Docker image, feel free to use it. The load balancer will accept HTTP/S connections from the clients and forward it to the farm of containers. All HTTP requests, from the client, will be redirected to HTTPS.

Let's get started!

### Requirements:

* AWS account - if you don't have one, you can create one [here](https://aws.amazon.com/).
* Familiarity with [AWS](httpts://aws.amazon.com), [Docker](https://www.docker.com/) and [PHP](https://www.php.net/) - *not required but a bonus*.
* Amazon Command Line Interface [CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
* A Docker image with a web server.

### What you'll do:

1. Build a Docker image.  See my tutorial on GitHub [here](https://github.com/ddella/PHP8-Nginx). If you have your own Docker image please feel free to use it. The image that I built has some function to return the IP address of the web server an some X-FORWARD-TO variables.
2. Push that image to Amazon ECR which is the equivalent of [Docker hub](https://hub.docker.com/).
3. Create two security groups, aka virtual firewalls. One for the load balancer and the other one for the containers.
4. Create a load balancer.
5. Create a service for Fargate to launch the tasks or containers.
6. Test Amazon ECS with your preferred web browser.

There's lots of steps but if you read and follow along you shouldn't have any problem to complete this workshop.

### Conventions:

Throughout this workshop, we will provide guidelines for you to configure the environment with the [AWS management console](https://console.aws.amazon.com). The only commands that requires the Amazon CLI are for Amazon ECR. These commands will look like this:
```sh
aws <command> <subcommand> [options and parameters]
```

Verify that Amazon CLI is installed with the following commands. As of writing this, on a macOS, the version is ```aws-cli/2.4.15 Python/3.8.8 Darwin/21.2.0 exe/x86_64 prompt/off```:
```sh
aws --version
```
### IMPORTANT: Workshop Cleanup

You will be deploying infrastructure on AWS which will have an associated cost.  Make sure everything is cleaned up when you're done with this workshop, to avoid unnecessary charges.

We're expecting the cost of this lab to be less than 1,00$ USD, even with free-tier.

## Let's Begin!

### AWS ECR - Create a private repository:

1. Open the Amazon ECR console and Click on **Get started**.
![Create Repository](images/Picture1.png)
2. Select **Private**, enter the **Repository name** ```ecr_repo_php8_nginx```  and hit **Create repository** at the bottom of the page.  
![Create Repository](images/Picture2.png)
3. On the main **Amazon ECR** page, select the new **Repository name** created in the preceeding step and click on the **View push commands**. The page will give you the information needed to **push** the image to ECR. It’s a four step process. If you followed my tutorial on building the Docker image, there’s no need to do step #2.
![Private repositories](images/Picture3.png)
![Push commands](images/Picture4.png)

### Push Docker image to AWS ECR:
This will copy the Docker image you have on your computer to the Amazon ECR repository you just created. I assume you followed the steps in my tutorial [here](https://github.com/ddella/PHP8-Nginx).  If not, just replace the name of the image.

Open a ```terminal``` for the next four steps.

1.Run the following command to retrieve an authentication token and authenticate your Docker client to your registry. Replace **012345678901** with your account id. If successful, you should get the the message: **Login Succeeded**
```sh
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 012345678901.dkr.ecr.us-east-2.amazonaws.com
```
2. Tag the Docker image you built to push it to your repository. Replace **012345678901** with your account id.
```sh
docker tag php8_nginx:latest 012345678901.dkr.ecr.us-east-2.amazonaws.com/ecr_repo_php8_nginx:latest
```
3. Check that the image has been tagged. Note that both images have the same **IMAGE ID**.
```sh
docker images
```
```
REPOSITORY  TAG  IMAGE ID  CREATED  SIZE
012345678901.dkr.ecr.us-east-2.amazonaws.com/php8_nginx  latest  b021dbb9e75e  20 minutes ago  30.4MB
php8_nginx latest  b021dbb9e75e  20 minutes ago  30.4MB
```
4. Run the following command to push this image to your newly created AWS ECR repository. Replace **012345678901** with your account id.
```sh
docker push 012345678901.dkr.ecr.us-east-2.amazonaws.com/php8_nginx:latest
```
5. Open the Amazon ECR Console and check that the container has been pushed. See the arrow to copy the **URI**, you will need it later.
![Container URI](images/Picture5.png)

### Checkpoint:
At this point, you should have a Docker image stored in your ECR private repository. Select the image to see the detail.
![Container URI](images/Picture6.png)

## AWS Load Balancer
AWS load balancer automatically distributes the incoming traffic across multiple targets, such as EC2 instances, containers, and IP addresses, in one or more Availability Zones. It serves as the **single point** of contact for clients. The components of a load balancer are:
1. **Security Group** acts as a virtual firewall for your instance to control inbound and outbound traffic.
2. **Target Group** routes requests to one or more registered targets, such as EC2 instances, containers, using the protocol and port number that you specify.
3. **Listeners** checks for connection requests from clients, using the protocol and port that you configure.


### Create a Security Group for the Load Balancer:

In this section we'll create a security group for the **load balancer**. The security group is a virtual firewall that sits in front of the load balancer. The ```inboud``` rules must permit all traffic for HTTP/S from the Internet. The table below shows all the ```inboud``` rules. You can change them to suit your needs. _We are not using IPv6 in this workshop even if we added the rules_.

|IPv4/6 |Protocol|TCP port|Source IP address|Description|
|:--- | :--- | :---:|---:|:---|
|IPv4| HTTP|80|0.0.0.0/0|Permit IPv4 HTTP|
|IPv6| HTTP|80|::/0|Permit IPv6 HTTP|
|IPv4| HTTPS|443|0.0.0.0/0|Permit IPv4 HTTPS|
|IPv6| HTTPS|443|::/0|Permit IPv6 HTTPS|

1. Open the Amazon **EC2** console, on the navigation pane, under **Network & Security** section select **Security Groups** and click on **Create security group**. Configure the **Name**, **Description** and your **VPC** where the security group will be used.
![Security Group1](images/Picture7.png)
2. Add the **Inbound rules**. Leave the default **Onbound rules**.
![Security Group1](images/Picture8.png)
3. Click **Create security group** at the bottom of the page.

### Create Target Group for the Load Balancer
In this section we create a ** Target Group** for the load balancer. For this workshop you need to use **IP addresses** as the target type.

1. Open the Amazon **EC2** console, on the navigation pane, under **Load Balancing** section, select **Target Groups** and click on **Create target group**. Configure the **Choose a target type**, this needs to be **IP** addresses. The protocol is HTTP port 80. Select the **VPC** where the containers will run. Leave the **Health checks** section with the default value and click **Next** at the bottom of the page.
![Target Groups1](images/Picture9.png)
2. Leave the rest with the default value. Click **Create target group** at the bottom of the page.
![Target Groups2](images/Picture10.png)
3. The status of the load balancer ```None associated```is normal . The load balancer association will be done in the next step.
![Target Groups3](images/Picture11.png)
### Create the Load Balancer
In this section we create the **Load Balancer** for our containers.

1. Open the Amazon **EC2** console, on the navigation pane, under **Load Balancing**, select **Load Balancers** and click on **Create Load Balancer**. Choose **Application Load Balancer** and click **Create**.
![Load Balancer Type](images/Picture12.png)
2. In the Basic configuration section, enter the Load balancer name and select **Internet-facing**.
![Basic Configuration](images/Picture13.png)
3. In the **Network mapping** section, select the same VPC as the **Target groups** and select the all the Availibility Zones where a container will be running. The load balancer routes traffic to targets, our containers, in these Availability Zones only.
![Network mapping](images/Picture14.png)
4. In the **Security groups** section, select the security group created for the load balancer. The load balancer only accepts connection on TCP port 80/443.
![Security Group](images/Picture15.png)
5. In the **Listeners and routing** section, configure two listeners, for HTTP and HTTPS. Choose the **target group**. The **target group** for HTTP will be changed later to ```redirect``` all traffic to HTTPS.
![Listeners and routing](images/Picture16.png)
6. In the **Secure listener settings** section, leave the **Security** policy to **ELBSecurityPolicy-2016-08**. This is the SSL Protocols/Ciphers. Select **Import** for the **Default SSL certificate**. If you used my Docker container, I have generated a **Certificate private key** (```nginx.key```) and a **Certificate body** (```nginx-certificate.crt```). Just copy/paste the file in the correct text box. When you’ll do the test at the end, your browser will complain about the self-signed certificate. That can be safely ignored. You can generate your own certificate and private key or use AWS certificate manager. Click **Create load balancer** at the bottom of the page. This will take a few minutes.
![Secure listener settings](images/Picture17.png)
7. Wait until you see a **State** of **active**.
![Load balancer state](images/Picture18.png)
#### Listeners
In step #5 of the preceding section, we mentioned that we needed to change the **Listeners**. The goal is to redirect all HTTP traffic, from clients, to HTTPS and send this traffic to the containers. This is a simple ```HTTP response status code 301``` sent by the load balancer to clients, if they used HTTP.

1. In the **Listeners** tab of the **Load Balancers** page, we need to modify the **listener**. Select the **HTTP:80** and click **Edit**.
![Edit listener](images/Picture19.png)
2. Remove the **Forward to**, click on **Add action** and select **Redirect**.
![Remove target group](images/Picture20.png)
3. Click on **Add action** and select **Forward**.
![Remove target group](images/Picture20-1.png)
4. Enter port **443** and leave the defaults, click **Save changes**.
![Redirect HTTP to HTTPS](images/Picture21.png)
5. The final listener rules are, every requests received on port **80** is redirected on port **443**. Every requests received on port **443** is directed on the target group **ECS-Target-Group-php8-nginx**.
![Listener rules](images/Picture22.png)
6. Make a copy the **DNS name**. This is the URL clients use to access your containers. It could also be configured as a ```CNAME```in **Route 53**, if you have your own domain.
![DNS name](images/Picture23.png)

### Create a security group for the Docker containers:
Create a security group for the containers. The only connection allowed on the containers  are from the load balancer. No traffic, directly from the Internet, is allowed to reach the containers. The security group needs to permit all ephemeral ```tcp``` port from load balancer. On AWS, the ephemeral port range for Elastic Load Balancers is ```0-65535```. _If you enter ```1024-65535```, your containers won’t be reachable._

|IPv4/6 |Protocol|TCP port|Source IP address|Description|
|:--- |:--- |:---:|:---:|:--- |
|Custom TCP| TCP|0-65535|Load balancer SG|Permit traffic from load balancer|

1. Open the Amazon **EC2** console, on the navigation pane, in the **Network & Security** section select **Security Groups** and Click on **Create security group**. Configure the **Name**, **Description** and your **VPC** where the load balancer will sit.
![Basic detail](images/Picture24.png)
2. Add the **Inbound rules**.The Docker containers need to accept traffic from the load balancer only. Select **All TCP** for the **Type** the **Port range** is automatically filled with ```0-65535```. The **Source** needs to be the ```load balancer Security Group```. Click on **Save rules** Leave the default **Onbound rules**.
![Inbound rules](images/Picture25.png)
3. Click **Create security group** at the bottom of the page.

## Create an ECS service
### Definitions
**CLUSTER**: In the context of **EC2**, it’s a logical group of Amazon EC2 virtual machine. In the context of **Fargate**, it’s a group of tasks which is a group of containers, see below.
**SERVICES**: A scheduling logic that decides when **Tasks** should run and that the amount of configured tasks are running. It’s optional, you don’t need to configure **Services** but in this workshop, we'll configure a **Service**.
**TASKS**: A logical group of running containers (the running instances).
**TASK DEFINITIONS**: Templates for your tasks. This is where you specify the Docker image, memory/cpu, VPC, …
**CONTAINERS**: Virtualized instance that you run your Tasks.

### Create task definition
1. Open the Amazon **ECS** console, on the navigation pane, under the **Amazon ECS** section select **Task Definitions** and Click on **Create new Task Definition**. _As of early 2022, the **New ECS Experience** doesn't work_, use the old one.
![Create task definition](images/Picture26.png)
2. Select **Fargate** as the launch type and click **Next**.
![Create task definition](images/Picture27.png)
3. In the Configure task and container definitions, enter the Name, the Task role and the operating system of the container.
![Task and container definition](images/Picture28.png)
4. In the **Task execution IAM role**, the box should already been filled. For the **Task size**, choose the minimum. It will be sufficient for this workshop. Click the **Add container** button. This will open a window on the right hand side for the container information.
![Task size](images/Picture29.png)
5. You will need the container **URI**. If you didn't copy it, duplicate the browser tab, go back to the **ECR Repositories** page and copy the **URI** in your clipboard. Close the duplicate tab you just opened.
![Container URI](images/Picture30.png)
6. Enter the **Container name**, paste the **Image URI** and set the **Port mappings**.
![Add container](images/Picture31.png)
7. In the Advanced container configuration unselect the **Log configuration** in the **STORAGE AND LOGGING** section almost at the bottom. This is the **Auto-configure CloudWatch Logs**.
![CloudWatch](images/Picture32.png)
8. Leave all the other fileds at their default and click **Add**.You are back at the **Configure task and container definitions** page.
![Add container](images/Picture33.png)
9. Leave the rest at their default value and click **Create**.

### Create a cluster
1. Open the Amazon **ECS** console, on the navigation pane, in the **Amazon ECS** section, select **Clusters** and Click on **Create Cluster**.
![Create Clusters](images/Picture34.png)
2. Select the **Network only** template and click **Next step**.
![Template](images/Picture35.png)
3. Enter the **Cluster name** and click **Create**.
![Cluster name](images/Picture36.png)
4. You should get the following page, click **View Cluster**.
![View Cluster](images/Picture37.png)
### Create a service
We're almost there. The last thing is to create the **ECS Service**.

1. On the cluster page, select the **Services** tab and click **Create**.
![Create service](images/Picture38.png)
2. On the **Configure service**, fill out the fields. The **Number of tasks** is the number of container Fargate will run. We choosed to run three containers. They will be split in each of our three AZ. Click **Next step** at the bottom of the page.
![Configure service](images/Picture39.png)
3. The next page is the **Configure network**. This is the “tricky” portion. Select your **VPC** and your **Subnets** where the containers will run. The **Auto-assign public IP** needs to be **ENABLED** or your tasks won’t start, they will stay in **PENDING** state. Click **Edit** to edit your **Security groups**. It will open a side window.
![VPC](images/Picture40.png)
4. Select the **Security group ID** that was created for the containers and click **Save**.
![security groups](images/Picture40-1.png)
5. In the **Load balancing** section, select the **Application Load Balancer**. The **Load balancer name** will be filled automatically.
![Load Balancing](images/Picture41.png)
6. Just below you have to click **Add to load balancer**.
![Add Load Balancing](images/Picture42.png)
7. Click **Create new** in the **Target group name** box. Select the group **ECS-Target-Group-php8-nginx**. It should be the only one, if it's your first time with ECS.
![New Target group name](images/Picture43.png)
8. Select the **Target group name**, the **Production listener port** will be filled automaticly.
![Target group name](images/Picture43-1.png)
9. Leave the **Service Auto Scaling** to the default value and click **Next step**.
![Auto Scaling](images/Picture44.png)
10. Review and click **Create Service**. This will take some time, be patient. At this point you’re back to the main cluster page. Amazon Fargate will start three containers based on the image you pushed to Amazon ECR.
![Ccreate service](images/Picture44-1.png)
11. You can check that the **Tasks** are running.
![Cluster status page](images/Picture45.png)

### Checkpoint:
At this point, you should have running containers that are accessible through the load balancer.

[*^ back to top*](#Amazon-ECS-and-load-balancer-with-Docker-containers)

## Time to test the web servers
### Amazon Route 53
If you have a domain on **Route 53**, you can create a CNAME to the DNS name of the load balancer. If you don’t, you can register one for 3$-5$/year. _This step is optional_.
![Route 53](images/Picture46.png)
Record created.
![Create CNAM](images/Picture47.png)

### Web Browser
Open you browser and type [http://<url>](http://%3curl%3e) or [https://<url>](https://%3curl%3e). Click on reload the page and the IP address of the server should ```round robin``` between the three containers created by Fargate.
![Web browser](images/Picture48.png)

The ```url``` is the load balancer ```DNS name``` or the Amazon Route 53 ```CNAME```, if you created one.

## Workshop Cleanup

This is really important because if you leave stuff running in your account, it will continue to generate charges.  Certain things were created manually throughout this workshop.  Follow the steps below to make sure you clean up properly.

Delete manually created resources throughout the labs:

* ECS service - Delete the ECS service itself.
* ECS service - Stop all the tasks.
* ECR - Delete any Docker images pushed to your ECR repository.
* EC2 ALB - Delete the load balancer and associated target groups.
* Route 53 - delete the ```CNAME```, if you created one.

# [CHANGELOG](./CHANGELOG.md)

# License
This project is licensed under the [MIT license](LICENSE).

[*^ back to top*](#Amazon-ECS-and-load-balancer-with-Docker-containers)
