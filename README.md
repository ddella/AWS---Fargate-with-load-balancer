
# Amazon ECS and load balancer with Docker containers

In this lab, you'll build a Docker image with Nginx web server and PHP 8 and deploy it on Amazon ECS with a load balancer. If you already have your Docker image, feel free to use it. The load balancer will accept HTTP/S connections. All HTTP requests will be redirected to HTTPS.

Let's get started!

### Requirements:

* AWS account - if you don't have one, you can create one [here](https://aws.amazon.com/).
* Familiarity with [AWS](httpts://aws.amazon.com), [Docker](https://www.docker.com/) and [PHP](https://www.php.net/) - *not required but a bonus*.

### What you'll do:

1. Build a Docker image.  See my tutorial on GitHub [here](https://github.com/ddella/PHP8-Nginx). If you have your own Docker image please feel free to use it. The image that I built has some function to return the IP address of the web server an some X-FORWARD-TO variables.
2. Push that image to AWS ECR. This is the equivalent to [Docker hub](https://hub.docker.com/). 
3. Create two security groups. One for the load balancer and the other one for the containers.
4. Create a load balancer.
5. Create a service for Fargate to launch the tasks or containers.
6. Last you open ypur browser and enter the URL of the load balancer.

There's lots of steps but if you read and follow along you shouldn't have any problem to complete this workshop.

### Conventions:

Throughout this workshop, we will provide commands for you to run in the [AWS management console](https://console.aws.amazon.com).

### IMPORTANT: Workshop Cleanup

You will be deploying infrastructure on AWS which will have an associated cost.  Make sure everything is cleaned up when you're done with this workshop, to avoid unnecessary charges.

We're expecting the cost of this lab to be less than 1,00$USD, even with free-tier.

## Let's Begin!

### AWS ECR - Create a repository:

1. Open the Amazon ECR console and Click on **Get started**.
![Create Repository](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture1.png)
2. Select **Private**, enter the **Repository name** ```ecr_repo_php8_nginx```  and hit **Create repository** at the bottom of the page.
![Create Repository](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture2.png)
3. On the main **Amazon ECR** page, select the new **Repository name** created in the preceeding step and click on the **View push commands**. The page will give you the information needed to **push** the image to ECR. It’s a four step process. If you followed my tutorial on building the Docker image, there’s no need to do step #2.
![Private repositories](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture3.png)
![Push commands](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture4.png)

### Push Docker image to AWS ECR:

1.Run the following command to retrieve an authentication token and authenticate your Docker client to your registry. **Replace 012345678901 with your account id**. If successful, you should get the the message: **Login Succeeded**
```sh
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 012345678901.dkr.ecr.us-east-2.amazonaws.com
```
2. Tag the Docker image you built to push it to your repository. **Replace 012345678901 with your account id**.
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
4. Run the following command to push this image to your newly created AWS ECR repository. **Replace 012345678901 with your account id**.
```sh
docker push 012345678901.dkr.ecr.us-east-2.amazonaws.com/php8_nginx:latest
```
5. Open the Amazon ECR Console and check that the container has been pushed. See the arrow to copy the **URI**, you will need it later.
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture5.png)
### Checkpoint:
At this point, you should have a Docker image stored in your ECR private repository.

Select the image to see the detail.
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture6.png)
## AWS Load Balancer
AWS load balancer automatically distributes the incoming traffic across multiple targets, such as EC2 instances, containers, and IP addresses, in one or more Availability Zones. It serves as the **single point** of contact for clients. The components of a load balancer are:
1. **Security Group** acts as a virtual firewall for your instance to control inbound and outbound traffic.
2. **Target Group** routes requests to one or more registered targets, such as EC2 instances, using the protocol and port number that you specify.
3. **Listeners** checks for connection requests from clients, using the protocol and port that you configure.


### Create a Security Group for the Load Balancer:

In this section we need to create a security group for the **load balancer**. The security group is a firewall that sits in front of the load balancer. The ```inboud``` rules must permit all traffic for HTTP/S from the Internet. Following are the rules. You can change them to suit your needs. _We are not using IPv6 in the lab_.

|IPv4/6 |Protocol|TCP port|Source IP address|Description|
|:--- | :--- | :---:|---:|:---|
|IPv4| HTTP|80|0.0.0.0/0|Permit IPv4 HTTP|
|IPv6| HTTP|80|::/0|Permit IPv6 HTTP|
|IPv4| HTTPS|443|0.0.0.0/0|Permit IPv4 HTTPS|
|IPv6| HTTPS|443|::/0|Permit IPv6 HTTPS|

1. Open the Amazon **EC2** console, on the navigation pane, under **Network & Security** section select **Security Groups** and click on **Create security group**. Configure the **Name**, **Description** and your **VPC** where the security group will be used.
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture7.png)
2. Add the **Inbound rules**. Leave the default **Onbound rules**. 
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture8.png)
3. Click **Create security group** at the bottom of the page.

### Create Target Group for the Load Balancer
In this section we create a ** Target Group** for the load balancer. In our case, the target type is **IP addresses**.

1. Open the Amazon **EC2** console, on the navigation pane, under **Load Balancing** section, select **Target Groups** and click on **Create target group**. Configure the **Choose a target type**, this needs to be **IP** addresses. The protocol is HTTP port 80. Select the **VPC** where the containers will run. Leave the **Health checks** section with the default value and click **Next** at the bottom of the page.
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture9.png)
2. Leave the rest with the default value. Click **Create target group** at the bottom of the page.
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture10.png)
3. Don't worry about the fact that no load balancer . The load balancer association will be done in the next step.
![Target Groups](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture11.png)
### Create the Load Balancer
In this section we actually create the **Load Balancer** for our containers.

1. Open the Amazon **EC2** console, on the navigation pane, under **Load Balancing**, select **Load Balancers** and click on **Create Load Balancer**. Choose **Application Load Balancer** and click **Create**.
![Load Balancer Type](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture12.png)
2. In the Basic configuration section, enter the Load balancer name and select **Internet-facing**.
![Basic Configuration](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture13.png)
3. In the **Network mapping** section, select the same VPC as the **Target groups** and select the all the Availibility Zones where a container will be running. The load balancer routes traffic to targets, our containers, in these Availability Zones only.
![Network mapping](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture14.png)
4. In the **Security groups** section, select the security group created for the load balancer. The load balancer only accepts connection on TCP port 80/443.
![Security Group](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture15.png)
5. In the **Listeners and routing** section, configure two listeners, for HTTP and HTTPS. Choose the **target group**. The **target group** for HTTP will be changed later to ```redirect``` all traffic to HTTPS.
![Listeners and routing](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture16.png)
6. In the **Secure listener settings** section, leave the **Security** policy to **ELBSecurityPolicy-2016-08**. This is the SSL Protocols/Ciphers. Select **Import** for the **Default SSL certificate**. If you used my Docker container, I have generated a **Certificate private key** (```nginx.key```) and a **Certificate body** (```nginx-certificate.crt```). Just copy/paste the file in the correct text box. When you’ll do the test at the end, your browser will complain about the self-signed certificate. That can be safely ignored. You can generate your own certificate and private key or use AWS certificate manager.
![Secure listener settings](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture17.png)
7. Click **Create load balancer** at the bottom of the page. This will take a few minutes. Wait until you see a **State** of **active**.
![Load balancer state](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture18.png)
#### Listeners
In step #5 of the preceding section, we mentioned that we needed to change the **Listeners**. The goal is to redirect all HTTP traffic, from client, to HTTPS and send this traffic to the containers. This is a simple ```HTTP response status code 301``` sent by the load balancer to client if they use HTTP.

1. In the **Listeners** tab of the **Load Balancers** page, we need to modify the **listener**. Select the **HTTP:80** and click **Edit**.
![Edit listener](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture19.png)
2. Remove the **Forward to**, click on **Add action** and select **Redirect**.
![Remove target group](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture20.png)
3. Enter port **443** and leave the defaults, click **Save changes**.
![Redirect HTTP to HTTPS](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture21.png)
4. The final listener rules are, every request received on port **80** is redirected on port **443**. Every request received on port **443** is directed on the target group **ECS-Target-Group-php8-nginx**.
![Listener rules](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture22.png)
5. Make a copy the **DNS name**. This is the URL clients use to access your containers. It could also be configured as a ```CNAME```in **Route 53** if you have your own domain.
![DNS name](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture23.png)
### Create a security group for the Docker containers:
Create a security group for the containers. The only connection allowed on the containers  are from the load balancer. No traffic, directly from the Internet, is allowed to reach the containers. The security group needs to permit all ephemeral tcp port from load balancer. On AWS, the ephemeral port range for Elastic Load Balancers is 0-65535. _If you enter 1024-65535, your containers won’t be reachable._

|IPv4/6 |Protocol|TCP port|Source IP address|Description|
|:--- |:--- |:---:| ---:|:--- |
|Custom TCP| TCP|0-65535|Load balancer SG|Permit traffic from load balancer|

1. Open the Amazon **EC2** console, in the **Network & Security** section select **Security Groups** and Click on **Create security group**. Configure the **Name**, **Description** and your **VPC** where the load balancer will sit.
![Basic detail](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture24.png)
2. Add the **Inbound rules**.The Docker containers need to accept traffic from the load balancer only. Select **All TCP** for the **Type** the **Port range** is automatically filled with ```0-65535```. The **Source** needs to be the ```load balancer Security Group```. Click on **Save rules** Leave the default **Onbound rules**.
![Inbound rules](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture25.png)
3. Click **Create security group** at the bottom of the page.

## Create an ECS service
### Definitions
**CLUSTER**: In the context of **EC2**, it’s a logical group of Amazon EC2 virtual machine. In the context of **Fargate**, it’s a group of tasks which is a group of containers, see below.
**SERVICES**: A scheduling logic that decides when **Tasks** should run and that the amount of configured tasks are running. It’s optional, you don’t need to configure **Services**.
**TASKS**: A logical group of running containers (the running instances).
**TASK DEFINITIONS**: Templates for your tasks. This is where you specify the Docker image, memory/cpu, VPC, …
**CONTAINERS**: Virtualized instance that you run your Tasks.

### Create task definition
1. Open the Amazon **ECS** console, on the navigation pane, under the **Amazon ECS** section select **Task Definitions** and Click on **Create new Task Definition**.
![Create task definition](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture26.png)
2. Select **Fargate** as the launch type and click **Next**.
![Create task definition](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture27.png)
3. In the Configure task and container definitions, enter the Name, the Task role and the operating system of the container.
![Task and container definition](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture28.png)
4. In the **Task execution IAM role**, the box should already been filled. For the **Task size**, choose the minimum for the lab. Click the **Add container** button. This will open a window on the right hand side for the container information.
![Task size](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture29.png)
5. You will need the container **URI**. Duplicate the browser tab, go back to the **ECR Repositories** page and copy the **URI** in your clipboard. Close the duplicate tab you just opened.
![Container URI](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture30.png)
6. Enter the **Container name**, paste the **Image URI** and set the **Port mappings**.
![Add container](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture31.png)
7. In the Advanced container configuration unselect the **Log configuration** in the **STORAGE AND LOGGING** section almost at the bottom. This is the **Auto-configure CloudWatch Logs**.
![CloudWatch](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture32.png)
8. Leave all the other fileds at their default and click **Add**.You are back at the **Configure task and container definitions page**.
![Add container](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture33.png)
9. Leave the rest at their default value and click **Create**.

### Create a cluster
1. Open the Amazon **ECS** console, on the navigation pane, in the **Amazon ECS** section, select **Clusters** and Click on **Create Cluster**.
![Create Clusters](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture34.png)
2. Select the **Network only** template and click **Next step**.
![Cluster template](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture35.png)
3. Enter the **Cluster name** and click **Create**.
![Cluster name](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture36.png)
4. You should get the following page, click **View Cluster**.
![View Cluster](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture37.png)
### Create a service
We're almost there. The last thing is to create the **ECS Service**.

1. On the cluster page, select the **Services** tab and click **Create**.
![Create service](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture38.png)
2. On the **Configure service**, fill out the fields. The **Number of tasks** is the number of container Fargate will run. Click **Next step** at the bottom of the page.
![Configure service](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture39.png)
3. The next page is the **Configure network**. This is the “tricky” portion. Select your **VPC** and your **Subnets** where the containers will run. The **Auto-assign public IP** needs to be **ENABLED** or your tasks won’t start. The will stay in **PENDING** state after you create your service later.
![VPC and security groups](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture40.png)
4. In the **Load balancing** section, select the **Application Load Balancer**. The name should be filled automatically.
![Load Balancing](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture41.png)
5. Just below you have to click **Add to load balancer**.
![Add Load Balancing](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture42.png)
6. New fields will appear, select the **Target group** in the **Target group name**. Click **Next step**.
![Containers](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture43.png)
7. Leave the **Service Auto Scaling** to the default value and click **Next step**.
![Auto Scaling](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture44.png)
8. Review and click **Create Service**. This will take some time. At this point you’re back to the main cluster page. You can check the **Tasks** and you should see three.
![Cluster status page](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture45.png)
  
### Checkpoint:
At this point, you should have a working container for the monolith codebase stored in an ECR repository and ready to deploy with ECS in the next lab.

[*^ back to top*](#Amazon-ECS-and-load-balancer-with-Docker-containers)

## Time to test the web servers
### Amazon Route 53
If you have a domain on **Route 53**, you can create a CNAME to the DNS name of the load balancer. If you don’t, you can register one for 3$-5$/year. This step is optional.
![Route 53](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture46.png)

### Web Browser
Open you browser and type [http://<url>](http://%3curl%3e) or [https://<url>](https://%3curl%3e). Click on reload the page and the IP address of the server should “round robin” between the three containers created by Fargate.
![Route 53](file:///Users/daniel/Training/AWS/AWS%20Labs/ECS/Fargate/images/Picture47.png)

The ```url``` is the load balancer ```url``` or the Amazon Route 53, if you decided to create a ```CNAME```.

## Workshop Cleanup

This is really important because if you leave stuff running in your account, it will continue to generate charges.  Certain things were created manually throughout this workshop.  Follow the steps below to make sure you clean up properly.

Delete manually created resources throughout the labs:

* ECS service - first update the desired task count to be 0.  Then delete the ECS service itself.
* ECR - delete any Docker images pushed to your ECR repository.
* ALBs and associated target groups

[*^ back to top*](#Amazon-ECS-and-load-balancer-with-Docker-containers)

