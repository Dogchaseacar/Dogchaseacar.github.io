***First of all， I would like to pay my highest respect and appreciation to a friend from the other side of the earth,  whose name is Shell845.  Although we haven't met each other yet,  she has generously given me precious advice on my project.   Here I wanna say thank you so much!***

<font size=3>OK, let's go back to the deployment notes. The target of this blog is taking down the steps and some problems I've came across during the deployment. I am deploying my java app on AWS, so first we need to apply for an AWS account, fortunately, AWS is now providing free trial in one year with limited function. But it is enough for us to deploy our own blog on it.</font>

## TOOLs
##### AWS EC2
##### AWS RDS
##### AWS Route 53
##### Tomcat
##### Nginx

## Step 1 Create EC2 instance
<font size=3>After signing in your AWS account, choose the EC2 service and create an EC2 instance. I am using Linux X86_64 20GB free tier. Carefully keep the master user name and password you have set. Except the security groups, just use the default settings AWS provides.

Set the inbound rules of your security group as follows:</font>

| Type        | protocal   |  Port range  |Source |
| --------   | -----:  | :----:  |:----:  |
| SSH      | TCP   |  22   |0.0.0.0/0|
| Custom TCP|  TCP  |   8080  |0.0.0.0/0|
| MYSQL/Aurora|   TCP    | 3306 |the source of your db|

<font size=3>We need SSH port for connecting to. So if you want you can also fix the source on your own IP.
Port 3306 is for the RDS where we will store our data.
8080 is for the website access.
Then after setting, you will get the key pair file. Carefully keep it, it's necessary for us to connect to the this instance!

After creatiing the EC2 instance, it will take few minutes to initialize. After it is done. We can SSH to this server!!! This EC2 instance will be used to hold our java app.</font>

## Step 2 Create AWS RDS
<font size=3>We want our data base distributedly deployed to make it safer and stable. We need to apply for an AWS RDS to store our data of our app. I am using MySQL, you can choose any other database you'd like to. After creating, set the RDS to the same VPC as our linux EC2 instance. The seurity group is as follows:</font>

| Type        | protocal   |  Port range  |Source |
| --------   | -----:  | :----:  |:----:  |
| SSH      | TCP   |  22   |the source of your linux EC2 instance|

<font size=3>Don't forget:  SSH to conncect to your RDS and create the databases you need.</font>

## Step 3 Install jdk and Tomcat
<font size=3>In the server we just created we need jdk to run our java app and Tomcat. So we must install the jdk first.
Use the ` wget ` command to download the jdk from oracle. Here is a important tip you need remember. If you directly copy the link from the website you can only download the HTML not the jdk. That is because the oracle has set an authentication process for downloading. The correct trick is first downloading the jdk on your own PC after doing the authentication, copy the link from downloader of your browser. Then use this link for your `wget` command instead of the previous one. This is a very hidden but useful trick for downloading jdk from oracle.

After downloading, Use `tar zxvf jdk-8u131-linux-x64.tar.gz` command to unzip the file you just downloaded, the name of the file might be different, just simply change it to the file name you just downloaded.

After unzipping, don't forget to  add the environmental variables. `sudo vim /etc/profile` add the following text to this profile</font>

	JAVA_HOME=/home/java/jdk1.8.0_131
	JRE_HOME=$JAVA_HOME/jre
	PATH=$PATH:$JAVA_HOME/bin
	CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
	export JAVA_HOME
	export JRE_HOME
	export PATH
	export CLASSPATH
Now the jdk install is finished.

<font size=3>Next is to install Tomcat. This time, you can simply copy the download link from Tomcat official wevsite. And use `wget downloadlink`, after downloading unzip `tar zxvf tomcatfilename.tar.gz`.

Now `cd tomcatpath/bin` input `./startup.sh`, you can start up the Tomcat! Now you can check if it is successfully installed by accessing to http://EC2IP:8080 in browser. If you can see the default page of Tomcat it means that everything is correct!</font>

## Step 4 Wrap up your java app and upload to server
<font size=3>Before wrapping up to war, you'd better make sure two things, one is adding `<packaging>war</packaging>` to your pom.xml. The other is making sure that your application file is correctly chosen and pointing to the AWS RDS. I've made some mistakes here and my time was wasted!
After checking these two things, cd to your java app path, then `mvn clean package -Dmaven.test.skip=true`. After it is done, you will get your war file. Then you can upload your war file to your server by `scp -i <your keypair path> <your java app war path> ec2-user@<yourserverIP>:/home/ec2-user/apache-tomcat-9.0.41/webapps`.

SSH to your linux EC2, cd to the webapps file, rename the ROOT file to another name just in case. If you made some mistakes you can still rename it back to help you debug. The cd to the bin file of Tomcat. Restart Tomcat, because your java app will be nuzipped by starting Tomcat. Then cd back to webapps, rename your java app to ROOT. This is because Tomcat will run the ROOT file by default on 8080. If nothing is wrong, you can browse your own java web by http://EC2IP:8080!</font>

## Step 5 DNS reflect
<font size=3>We don't want our friends to browse our website via raw IP. So we need do this DNS process. First by a domain from a domain seller website and verify it. Then create a public hosted zone on AWS Route 53 with your domain name. Then add two A records. And choose Alias on the www.name.com one.

| Record name       | Type  |  Value  |
| --------   | -----:  | :----:  |
| domainname.com      |A  |  your EC2IP  |
| www.domainname.com      |A  |  domainname.com  |

<font size=3>Then copy the value of NS record. Open the management page of your domain seller, paste the value to the custom DNS server. The we need wait for while, it really depends, usually 1 day to 2days util our domain can be accessed. After that you can browse our own web via http://yourdomainname:8080.</font>

## Step 6 Reverse proxy by Nginx

<font size=3>Adding the port at the end of your domain is still annoying and not what we want. In addition we want to increase the browsing speed and load balancer. We need the help of reverse proxy. It is quite convenient to do this by Nginx.

First create another ubuntu instance on EC2 to hold Nginx. Then follow the guidance from official website http://nginx.org/en/linux_packages.html to install Nginx.

After installation, `cd /etc/nginx/nginx.conf` edit the `niginx.conf` by vim.
add your own server block to it.</font>

	server {
	listen      80;
	server_name yourdomain.com www.yourdomain.com;

	location / {
	proxy_pass http://yourjavaapphostIP:8080;
	}
	}

<font size=3>Then go back to Route 53, change the value of the domainname.com to the Ip of your ubuntu server.
Done! Now you can access to your web without 8080!!!!</font>

## Step 7 Add HTTPS to your web

<font size=3>To make your website safer, you can use Cerbot to make it, which I also learned from Shell845 XD~
Here is the link of the guidance which is qiute clear and convenient.</font>

https://certbot.eff.org/lets-encrypt/ubuntufocal-nginx.