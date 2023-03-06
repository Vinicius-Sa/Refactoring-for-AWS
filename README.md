#REFACTORING WITH AWS 

The objective of this project is re-architect services for the AWS cloud  

Instead of using infrastructure as a service, we will be using mostly PASS and SAAS services so AWS will manage services for us, the infra will be on code, so refactoring our application gives us an easy infrastructure to manage, good performance, convenient to scale, and no need huge teams to manage all this
 

**AWS SERVICES**
> Front-end
- BEANSTALK - EC2 for Tomcat
- BEANSTALK - Auto-scaling 
- BEANSTALK - Continous delivery 
- CloudFront - Delivery content network 
- RDS - Mysql DB 
- S3/EFS - Storage 

> Back-end 
- RDS - Mysql DB 
- Elastic cache - replacing MemcacheD
- Active MQ - replacing Rabbit MQ
- Route 53 - DNS



**COMPLIANCE** 
- Flexible infra
- No upfront cost
- IAC 
- PAAS & SAAS 
- Ease infra management  

SETUP  
- So first create Keypairs and Security group, in sg is important to create a rule allowing traffic for himself so with that our backend services can communicate with each other 
- Create the subnet group with the subnets that we want to deploy DB instances, then create a parameter group specifying the engine of the Database of your choice, customize the PG configs of your preference, and finally create RDS DB it could be Aurora, MySQL or MariaDB in this project I'm gonna use MySql
- Next, let's create an Elasticache parameter group with family memcached1.4, also create a subnet group, then create a Memcached cluster, make sure to select the right parameter group, subnet group, and backend security group
- Now Amazon MQ, the engine we will go with rabbitMQ, single instance, engine version 3.9, make sure to mark the Private access option and select the backend security group 
- Launch a temporary instance just to install MySql client, log into the RDS instance, and initialize the database, make sure to create an sg for the instance, and on the backend sg allow traffic on 3306 for that sg, log into the database with **mysql -h <endpoint> -u admin -p<password>** check the tables, if everything fine lets initialize our database with our schema using **db_backup.sql** run **mysql -h mydbinstance.caq9xpxouqdi.us-east-1.rds.amazonaws.com -u admin -pvuecdCFgNSExzKW49Uh0 accounts < db_backup.sql** then check if the tables role, user, user_role are created 
- Now let's change the application.propetiers with our backend endpoint links 
- For the Elastic Beanstalk platform it will be Tomcat on version 8.5, go for more options to custom configuration, so Beanstalk gives us instances and load balancer, we are putting load balancer in a different sg and instance on the backend security group. On **modify capacity** is the configure capacity of your environment and auto-scaling settings to optimize the number of instances used, in my case I'm gonna use a minimum of 2 for high availability and a maximum of 4 instances, for scaling triggers we can use multiples metrics, network out is very popular for web applications but we could use CpuUtilization as well, for our purpose Network Out is the best so if the network traffic is going out on a larger amount, it will launch more instances. We also going to modify **rolling updates and deployments** this is one of the most interesting features of Beanstalk, when an artifact is deployed to our stack there are different policies that we can configure, like **all at once** so if you have ten instances, all the ten instances will be brought down and they will be upgraded, so, of course, there will be downtime.**Rolling** is better, so let's say you have ten instances and you specify percentage as 20 then two instances at a time will be upgraded, while the other ones handle the traffic, in my case I have two instances, so let's keep this as 50%. Now there are other deployment policies also, but they will have extra cost so **Rolling with additional batch**  that it's going to launch extra instances for the upgrade, **immutable** that it's going to launch the entire new stack so if you have ten instances, it's going to launch ten new instances, **Traffic splitting** is where you can route a certain percentage of traffic only to the newer version, so let's say 10% traffic goes to the new instances and the rest goes to the old instances, and then every 5 minutes, if it's good it will be increasing 10%, 10% until all instances, it's a safer option when you want to roll out something new to your user and you're not so sure how the user will react. Continuing, let's select the early created keypair so this is gonna be used to ssh on the Beasntalk instances
- Now that our Beanstalk is ready it should place the minimum instance running, the instances should be on Beanstalk SG and backend SG, so let's edit our backend SG to allow traffic from 3306, 11211, and 5671 to Beanstalk SG
- Next, let's go to our Beanstalk ALB and add listener HTTPS with our certificate, change the application health check path for /login, and select stickiness, so  if stickiness is enabled it will always connect to the same instance, and the current user who is logged in may lose the session, if it is not enabled, then it may route it to other instance also
- Build & Deploy the artifact, first update application.properties with the backend endpoints, username, and password for RDS, Amazon MQ, and Elastic Cache, make sure all the settings are right and build the artifact, then can upload the artifact on the application version and deploy 
- Next step is to create a record on your DNS provider pointing to our Beanstalk ALB endpoint, and now we can validate our stack 
- If we do not have clients around the world maybe CDN is not needed but I'm going to configure a CloudFront delivery content network for our website data to be cached around the world, which reduces our client latency, so let's create a distribution, in origin domain name we are gonna enter our domain record, allow all HTTP methods, select our record on alternate domain name and the SSL certificate security policy TLSv1, protocol Match viewer so the user can access HTTP or HTTPS and create. Can validate if CloudFront is serving cache by looking at Cache statistics: 
![Captura de tela 2023-03-04 211157](https://user-images.githubusercontent.com/95035624/222935400-bcfd5c03-32ef-499e-b7be-9deae20a9b9a.png)

**ARCHITECTURE FLOW** 
User access Amazon Route 53 to Cloud Front then Application Load Balancer sends the request for one of our web services instances ( ALB & Instances auto-scaling is managed by Beanstalk), Monitored by CloudWatch alarms, artifacts are stored in S3 buckets. RDS MySql, Amazon MQ, and Elasticache instead of using MySQL-service, MemcacheD, and RabbitMQ from an instance.


![Captura de tela 2023-03-04 203608](https://user-images.githubusercontent.com/95035624/222935404-455d66b6-5696-4365-853f-6591309c705a.png)
![Captura de tela 2023-03-04 203305](https://user-images.githubusercontent.com/95035624/222935409-3735c4a0-2184-4c4c-8c6d-c39d082d8425.png)
![Captura de tela 2023-03-04 203342](https://user-images.githubusercontent.com/95035624/222935414-18bb6ffc-3c32-439e-8129-edd6e091e4ab.png)


