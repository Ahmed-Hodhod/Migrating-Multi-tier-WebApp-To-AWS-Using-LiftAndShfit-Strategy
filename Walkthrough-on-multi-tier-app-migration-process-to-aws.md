**In this walkthrough, we will migrate a multi-tier webapp [Legacy Java Application] to the cloud, namely AWS.**<br>
**We will run all our webapp environment components on EC2 instances fully managed by us.**<br>
**We will run our own Message Broker server, MySQL server, Java Application frontend server and Memcached server.**<br>

**_Through this journey, we are strictly implementing the Lift and Shift Migration strategy._**

## Prerequisite

You are excepted to be familiar with AWS compute services such as EC2. This is because AWS interface changes frequently and
sometimes we will have to go back and modify some properties to test the provisioned resources.

## AWS resources

**Please note that this project will incur some AWS charges. So, you can consider finishing the whole project in one go or stopping the instances
whenever you feel you need a break.**

- Elastic Cloud Compute(EC2)
- Elastic Load Balancer(ELB)
- Autoscaling Group(ASG)
- EFS/S3 for storage
- ROUTE 53
- Git Bash for windows users

## Workflow in a high context

- Users will be able to access the app using the ELB endpoint.
- ELB will forward the request to the tomcat server running on the app-instance (app-tier).
- The app-instance will access the mysql instance (database-tier) and rabbitMQ & memcached instances (backend-tier) through route 53 private hosted zone records.

| ![workflow](https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/workflow.png) |
| :--------------------------------------------------------------------------------------------------------------------------------: |
|                                                        The Workflow diagram                                                        |

**NOTE: Each tier will have a different security group.**

## Guidelines

First of all, you need to clone the source code [java application]:
`git clone -b aws-LiftAndShift https://github.com/devopshydclub/vprofile-project.git` <br>

Swtich to aws-LiftAndShift: `git checkout aws-LiftAndShift -f`<br>

### Prepare the infrastructure

#### 1. Create a security group for the ELB (vprofile-elb-sg)

The sg should allow any _http/https_ request from any ip4 or ipv6.<br>
_One of the best practices is to only allow https requests to the webapp._

#### 2. Create a security group for the app-tier (vrofile-app-sg)

The sg should only allow for traffic from vprofile-elb-sg on port 8080.<br>
The sg should also have port 22 open to be able to login to the instance and troubleshoot our running app.<br>

_our application is running on a tomcat server listening on port 8080._

#### 3. Create a security group for the app-tier (vrofile-backend-sg)

By backend, we mean the three instances on which memcached, rabbitMQ and mysql database will be running.

**_This sg should allow for traffic on the following ports:_**

- 3306 (mysql port) from vrofile-app-sg
- 11211 (MemCached port) from vrofile-app-sg
- 5672 (rabbitMQ port) from vrofile-app-sg
- 22 (SSH) from your IP.

The sg should also allow for inter-communication between the backend instances. To do that, you need to edit the vprofile-backend-sg
rules after saving the previous 3 rules. Choose All Traffic to allow communication on all ports and choose Source to be
vprofile-backend-sg.<br>

![backend-sg](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2203).png>)<br>

_NOTE: src/main/resources/application.properties (Java file) holds all the specifics about the app communication with other services._

#### 4. create a key-pair (.pem for git bash or .ppk for putty)

#### 5. Launch mysql instance

AMI: use centOS 7 from the marketplace <br>
sg: use vprofile-backend-sg <br>
userdata: copy-paste the script userdata/mysql.sh <br>

The userdata script is installing Mariadb server, creating a database (accounts) having a user (admin) with a password (admin123),
and finally populating the accounts database with some dummy data.

#### 7. (Validate) Login into the instance using git bash

- On EC2 dashboard, choose the app instance and connect. From SSH client tab, copy the SSH command and paste it to git bash to login to the instance.

![connect to the instance](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2207).png>)

- Switch to root: `sudo -i`
- Check whether mariadb server is running or not: systemctl status mariadb

In case of a failure, you can consider waiting for a while and then trying again because it may be taking time to execute the userdata script after launching the instance.

#### 8. Launch MemCached instance

AMI: use centOS 7 from the marketplace.<br>
sg: use vprofile-backend-sg <br>
userdata: copy-paste the script userdata/memcache.sh<br>

The script is installing the memcached server and starting it.

#### 9. Launch rabbitMQ instance

AMI: use centOS 7 from the marketplace <br>
sg: use vprofile-backend-sg <Br>
userdata: copy-paste the script userdata/rabbitmq.sh <br>

The script is installing rabbitMQ server & its dependencies and then starting it.

#### 10. In a blank file, write down the private IP of each of the 3 instances in the backend.

#### 11. Go to AWS Route 53 -> Create a private Hosted Zone

Domain Name: vprofile.in <br>
Region: us-east-1 (North Virginia) <br>
vpc: default<br>

![create a hosted zone](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2218).png>)

#### 12. Create simple records in the hosted zone

We will create three records for our backend instances.<br>
Record name: db01 <br>
endpoint: paste the IP copied from the blank file (step 9) <br>

Similarly, define the other two records.<br>
Now, we have 3 records as following:<br>

- db01.vprofile.in<br>
- mc01.vprofile.in<br>
- rmq01.vprofile.in<br>

This is one of the best practices regarding using AWS resources. Instead of hard-coding the public IP address of an instance,
you can rather use a Domain record. Also, public IP addresses are prune to change when restarting instances for any reason.

![sample zone records ](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2224).png>)

#### 13. Launch the app instance which is running on a tomcat server

AMI: Ubuntu
sg: use vprofile-app-sg
userdata: copy-paste the script userdata/tomcat_ubuntu.sh

### Build and Deploy Artifacts

#### 14. Open Git Bash

- clone the source code [skip this step if you already cloned the repo in the beginning]: `git clone -b aws-LiftAndShift https://github.com/devopshydclub/vprofile-project.git`
- swtich to aws-LiftAndShift: `git checkout aws-LiftAndShift -f`
- `vim vprofile-project/src/main/resource and update the application.properties` <br>
  instead of "db01" in the third line, make it: db01.vprofile.in<br>
  replace **db01** with `db01.vprofile.in`<br>
  replce **mc01** with `mc01.vprofile.in`<br>
  replace **rmq01** with `rmq01.vprofile.in`<br>

  ![application.properties](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2228).png>)

- Go back to vprofile-project where you have "pom.xml" file and execute this command to build the project and generate the artifact:<br> `mvn install`
- After it finishes executing, you should now have "vprofil-v2.war" in the newly created folder "target"

#### 15. create an S3 bucket and upload vprofile-v2.war ( artifact ) to it (use AWS console)<br>

You can also do that through the CLI (git bash). <br>You need to follow these steps:

- Create an IAM user with AmazonS3FullAccess policy attached to it
- Configure aws with this IAM user programmetic access keys: `aws configure`
- Create an S3 bucket: `aws s3 mb s3://vprofile-artifact-123456`
- Upload the artifact to the bucket: `aws s3 cp vprofile-v2.war s3://vprofile-artifact-123456/vprofile-v2.war`<BR>

_NOTE: the name of the bucket must be unique._ <br>
If you ran into a problem because of the name, you could suffix some random numbers to the bucket name to distinguish it.

#### 16. Create an IAM role and attach it to the app instance

- Create an IAM role with AmazonS3FullAccess
- On EC2 dashboard -> choose the app instance -> actions -> instance settings -> modify IAM role and attach the new role

#### 17. Move the artifact from the S3 bucket to EC2 app instance

- Login to app01 instance
- Make sure that tomcat server is running: `systemctl status tomcat8`

  ![checking the tomcat server status](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2211).png>)

- CD into: /var/lib/tomcat8/webapp#
- Stop tomcat server: `systemctl stop tomcat8`
- Remove the default application: `rm -rf ROOT`
- Install aws cli: `apt install awscli -y`<br>
  _NOTE: You don't need to configure aws here because the instance is assuming the role by default._<br>
- `aws s3 ls s3://vprofile-artifact-123456` (use your bucket name)
- Copy from S3 to EC2: `aws s3 cp s3://vprofile-artifact-123456/vprofile-v2.war /tmp/vprofile-v2.war` (use your bucket name)
- Move the artifact : `cp /tmp/vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war`
- Start the server again: `systemctl start tomcat8`
- List all folders under webapps, you should find ROOT and ROOT.war
- Check application.properties file: `cat /ROOT/WEB INF/classes/application.properties`<br>
- `apt install telnet`
- Check Connectivity to the backend instances: `telnet db01.vprofile.in 3306`<br>
  Please remember that vprofile-backend-sg allows requests from the vprofile-app-sg, so this should work fine.
- Till this point, you can already browse the webapp on `http://[the app-instance Public IP goes here]:8080`

#### 17. Now, it is the time to setup our loadbalancer

You need a target group for it with the following configurations:

- Health Check Path: /login
- Port: override to 8080
- Threshold: decrease to 2 or 3
- Register our app-instance -> include as pending

#### 18. Create an application load balancer

- select all zones
- select the loadbalancer sg
- Listener: 80

_NOTE: you can add a Listener on port 443 (https), but you will need to add an ACM certificate issued in the same region and this costs some dollars.<br>_
**_Please Note that the load balancer itself is listening on port 80 (http) while the target group is listening on port 8080 the same as tomcat server running on the app-instance._**

#### 19. Now, we are done with our migration process.

It is the time to test our running app on the ELB endpoint.<br>
Copy the ELB endpoint to the browser and login as admin_vp, password: admin_vp. You should be logged in and directed to the welcome page.

#### 20. Create an Autoscaling Group to handle high traffic

- Choose vprofile-app01 instance -> actions -> image -> create an image and give it a name.
  It will be available under AMIs on EC2 dashboard.
- Create a launch configuration using our AMI <Br>
  type: t2.micro <br>
  IAM instance profile: choose the role that we created for the app-instance <br>
  select vprofile-app-sg /keypair<br>

![creating a launch configuration](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2247).png>)

![creating a launch configuration-2](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2248).png>)

- On EC2 dashboard, Create ASG
  - Choose our launch configuration <br>
  - Select all the subnets to launch in any of the AZs<Br>
  - Enable loadbalancing and choose the loadbalancer<br>
  - MIN:1, Max: 4, Desired:1 <br>
  - Target Scaling policy: Target value = 50 <br>
  - Add Notification & Tags if you wish.<br>

![creating an ASG](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2249).png>)

![creating an ASG-2](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2250).png>)

#### 21. Test the ASG

You can test the Autoscaling Group by deleting the app-instance and waiting for some time. The ASG will launch an instance using the launch configuration.
<br>
_Please note that the number of instances that will be launched equals "Desired" and will never go below "Min" or above "Max"_<br>

| ![testing ASG](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2256).png>) |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|                                          An app-instance is being launched in response to deleting the app-instance.                                           |

| ![SNS topic](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2254).png>) |
| :----------------------------------------------------------------------------------------------------------------------------------------------------------: |
|                                     An email is received from the SNS topic when a new instance is launched by the ASG.                                      |

You can go further and test the ASG via running a Stress job [ Lookup how to do it using Stress package on linux].

| ![testing app](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2241).png>) |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|                                                Copy the endpoint of the ELB to the browser to view the webapp.                                                 |

| ![testing app](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2240).png>) |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|                                                           Access the webapp through teh ELB endpoint                                                           |

| ![Login to the app](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2243).png>) |
| :-----------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|                                                    Login to the webapp. <br> user: admin_vp , password: admin_vp                                                    |

| ![testing memcached-1](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2245).png>) |
| :--------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|                                          Quering the database for the first time. No data is cached in the memcached server.                                           |

| ![testing memcached-2](<https://github.com/Ahmed-Hodhod/Migrating-Multi-tier-WebApp-To-AWS-Using-LiftAndShfit-Strategy/blob/main/Screenshots/Screenshot%20(2246).png>) |
| :--------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|        Querying the database for the second time.<br> Notice that the query is not sent to the mysql server and the query result is fetched from the memcached.        |

## Conclusion

Lift and Shift strategy is the not the best way to migrate to the cloud. Cloud providers such as AWS provide lots of tools that could help the businesses to operate faster without heavy costs or the burdening management of the provisioned resources.
