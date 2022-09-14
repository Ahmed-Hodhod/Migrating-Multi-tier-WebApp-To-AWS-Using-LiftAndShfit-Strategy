In this walkthrough, we will migrate a multi-tier webapp to the cloud without changing the workflow [Lift and Shift Strategy].

## Prerequisite

You are excepted to be familiar with AWS compute services such as EC2. This is because AWS interface changes frequently and
sometimes we will have to go back and modify some properties to test the provisioned resources.

## AWS resources

Please note that this project will incur some AWS charges. So, you can consider finishing the whole project in one go or stopping the instances
whenever you feel you need a break.

Elastic Cloud Compute(EC2)
Elastic Load Balancer(ELB)
Autoscaling Group(ASG)
EFS/S3 for storage
ROUTE 53
ACM(to issue a certificate)
Git Bash for windows users

## Workflow in a high context

users with be able to access the app using ELB endpoint.
ELB will forward the request to the tomcat server (app-tier).
the app-tier will access mysql instance (database-tier) and rabbitMQ & memcached instances (backend-tier) through route 53 private hosted zone.

PS: each tier will have a different security group.

It is more or less the same local architecture, except for Nginx (load-balancer)
which we replaced with AWS ELB.

## Guidelines

1. Create a security group for the ELB (vprofile-elb-sg)  
   The sg should allow any http/https request from any ip4 or ipv6.
   One of the best practices is to only allow https requests to our webapp.

2. Createa a security group for the app-tier (vrofile-app-sg)
   The sg should only allow for traffic from vprofile-elb-sg on port 8080.
   The sg should also have port 22 open to be able to login to the instance and troubleshoot our running app.
   our application is running on a tomcat server listening on port 8080.

3. Create a security group for the app-tier (vrofile-backend-sg)  
   By backend we mean the three instances on which memcached, rabbitMQ and mysql database will be running.

   ##### This sg should allow for traffic on the following ports:

- 3306 (mysql port) from vrofile-app-sg
- 11211 (MemCached port) from vrofile-app-sg
- 5672 (rabbitMQ port) from vrofile-app-sg
- 22 (SSH) from your IP.

#

The sg should also allow for inter-communication between the backend instances. To do that, you need to edit the vprofile-backend-sg
rules after saving the previous 3 rules. Choose All Traffic to allow communication on all ports and choose Source to be
vprofile-backend-sg.
PS: src/main/resources/application.properties (Java file) holds all specifics about the app communication with other services.

#

4. create a key-pair (.pem for git bash or .ppk for putty)
   PS: we will use git bash in this setup.

#

5. Launch mysql instance
   AMI: use centOS 7 from the marketplace.
   sg: use vprofile-backend-sg
   userdata: copy-paste the script userdata/mysql.sh

The userdata script is installing Mariadb server, creating a database (accounts) having a user (admin) with a password (admin123),
and finally populating the accounts database with some dummy data.

#

7. (Validate) Login into the instance using git bash

- On instances dashboard, choose the app instance and connect. From SSH client tab, copy the SSH command and paste it to git bash to
  login to the instance.
- Switch to root: sudo -i
- Check whether mariadb server is running or not: systemctl status mariadb  
  In case of a failure, you can consider waiting for a while and then trying again because it may be taking time to execute the userdata
  script after launching the instance.

#

7. Launch MemCached instance
   AMI: use centOS 7 from the marketplace.
   sg: use vprofile-backend-sg
   userdata: copy-paste the script userdata/memcache.sh
   The script is installing the memcached server and starting it.

#

8. Launch rabbitMQ instance
   AMI: use centOS 7 from the marketplace.
   sg: use vprofile-backend-sg
   userdata: copy-paste the script userdata/rabbitmq.sh
   The script is installing rabbitMQ server & its dependencies and then starting it.

9. In a blank file, write down the private IP of each of the 3 instances in the backend.

#

10. Go to AWS Route 53 -> Create a private Hosted Zone  
    Domain Name: vprofile.in
    Region: us-east-1 (North Virginia)
    vpc: default

#

11. Create simple records in the hosted zone
    We will create three records for our backend instances.

Record name: db01
endpoint: paste the IP copied from the blank file (step 9)

Similarly, define the other two records.
Now, we have 3 records as following:
db01.vprofile.in
mc01.vprofile.in
rmq01.vprofile.in

This is one of the best practices regarding using AWS resources. Instead of hard-coding the public IP addresse of an instance,
you can rather use a Domain record. Also, public IP addresses are prune to change when restarting instances for any reason.

12. Launch the app instance which is running on a tomcat server  
    AMI: Ubuntu
    sg: use vprofile-app-sg
    userdata: copy-paste the script userdata/tomcat_ubuntu.sh

Build and Deploy Artifacts

1. Open Git Bash

- clone the source code: git clone -b aws-LiftAndShift https://github.com/devopshydclub/vprofile-project.git
- swtich to aws-LiftAndShift: git checkout aws-LiftAndShift -f
- cd into vprofile-project/src/main/resource and update the application.properties file:
  instead of "db01" in the third line, make it: db01.vprofile.in

db01 -> db01.vprofile.in
mc01 -> mc01.vprofile.in
rmq01 -> rmq01.vprofile.in

#

2. Go back to vprofile-project where you have "pom.xml" file and execute this command to build the project and
   generate the artifact: mvn install

#

3. After it finishes executing, you should now have "vprofil-v2.war" in the newly created folder "target"

#

4. create an S3 bucket and upload vprofile-v2.war ( artifact ) to it (use AWS console)
   You can also do that through the CLI (git bash). You need to follow these steps:

- Create an IAM user with AmazonS3FullAccess policy attached to it.
- Configure aws with this IAM user programmetic access keys.
- Create an S3 bucket: aws s3 mb s3://vprofile-artifact-123456
- Upload the artifact to the bucket: aws s3 cp vprofile-v2.war s3://vprofile-artifact-123456/vprofile-v2.war
  PS: the name of the bucket must be unique.
  If you ran into a problem because of the name, you could suffix some random numbers to the bucket name to distinguish it.

#

8. Create an IAM role and attach it to the app instance

- Create an IAM role with AmazonS3FullAccess
- On EC2 dashboard -> choose the app instance -> actions -> instance settings -> modify IAM role and attach the new role

#

9. Move the artifact from the S3 bucket to EC2 app instance

- Login to app01 instance
- Make sure that tomcat server is running: systemctl status tomcat8
- CD into: /var/lib/tomcat8/webapp#
- systemctl stop tomcat8
- rm -rf ROOT : the default application
- apt install awscli -y
  PS: You don't need to configure aws here because the instance is assuming the role by default.
- aws s3 ls s3://vprofile-artifact-123456 (use your bucket name)
- aws s3 cp s3://vprofile-artifact-123456/vprofile-v2.war /tmp/vprofile-v2.war (use your bucket name)
- cp /tmp/vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war
- systemctl start tomcat8
- List all folders under webapps, you should find ROOT and ROOT.war
- Check application.properties file: cat /ROOT/WEB INF/classes/application.properties
- apt install telnet
- Check Connectivity to the backend instances: telnet db01.vprofile.in 3306
  Please remember that vprofile-backend-sg allows requests from the vprofile-app-sg, so this should work fine.
- Till this point, you can already browse the webapp on http://[the app-instance Public IP goes here]:8080

#

10. Now, it is the time to setup our loadbalancer
    You need a target group for it with the following configurations:

- Health Check Path: /login
- Port: override to 8080
- Threshold: decrease to 2 or 3
- Register our app-instance -> include as pending

#

11. Create an application load balancer

- select all zones
- select the loadbalancer sg
- Listener: 80  
  PS: you can add a Listener on port 443 (https), but you will need to add an ACM certificate issued in the same region
  and this costs some dollars.
  Please Note that the load balancer itself is listening on port 80 (http) while the target group is listening on port 8080 the same as
  the tomcat server running on the app-instance.

#

12. Now, we are done with our migration process.
    It is the time to test our running app on the ELB endpoint.

- Copy the ELB endpoint to the browser and login as admin_vp, password: admin_vp
  You should be logged in and directed to the welcome page.

#

13. Create an Autoscaling Group to handle high traffic

- Choose vprofile-app01 instance -> actions -> image -> create an image and give it a name.
  It will be available under AMIs on EC2 dashboard.
- Create a launch configuration using our AMI
  type: t2.micro ->
  IAM instance profile: choose the role that we created for the app-instance  
  select vprofile-app-sg /keypair
- On EC2 dashboard, Create ASG
  Choose our launch configuration.
  Select all the subnets to launch in any of the AZs.
  Enable loadbalancing and choose the loadbalancer.
  MIN:1, Max: 4, Desired:1 .
  Target Scaling policy: Target value = 50 .
  Add notification, tags if you wish.

#

14. You can test the Autoscaling Group by deleting the app-instance and waiting for some time. The ASG will launch an instance
    using the launch configuration.
    Please note that the number of instances that will be launched equals "Desired" and will never go below "Min" or above "Max"

You can go further and test the ASG via running a Stress job [ Lookup how to do it using Stress package on linux].

## Conclusion

In this walkthrough, we shifted the project manually to EC2 instances managed by us on AWS using the classic approach [Simple Lift & Shift]
