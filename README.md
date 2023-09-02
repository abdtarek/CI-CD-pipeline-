# CI-pipeline-using jenkns, sonarqube, Nexus and Slack
## What is continuous integration in Jenkins?
Continuous Integration is a software development process where a code is continuously tested after a commit to ensure there are no bugs. In large teams, many developers work on the same code base. Thus, any of the multiple commits can have a bug.
## Pre-requisites:
- AWS Account
- GitHub account
- Jenkins
- Nexus
- SonarQube
- Slack
  
![image](https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/3d3d2fbf-ae42-4d8c-8a21-9b4e3fcb2591)

## step-1 creating the key pair

To connect to your EC2 instances, you will need to create a key pair. A key pair is a set of two files: a public key and a private key. The public key is stored on your local computer, and the private key is stored on the EC2 instance.

To create a key pair, open the AWS Management Console and go to the EC2 dashboard. In the navigation pane, select Key Pairs. Then, click Create Key Pair.

In the Key Pair Name field, enter a name for your key pair. Then, click Create.

The AWS Management Console will download the private key file to your local computer. Save the file in a secure location. You will need this file to connect to your EC2 instances.

## Step-2: Create Security Groups for Jenkins, Nexus and SonarQube

```
Name: jenkins-SG
Allow: SSH from MyIP
Allow: 8080 from Anywhere IPv4 and IPv6 (We will create a Github webhook which will trigger Jenkins)
```
```
Name: nexus-SG
Allow: SSH from MyIP
Allow: 8081 from MyIP and Jenkins-SG
```
```
Name: sonar-SG
Allow: SSH from MyIP
Allow: 80 from MyIP and Jenkins-SG
```
- Once we created sonar-SG, we will add another entry to jenkins Inbound rule as Allow access on 8080 from sonar-SG. SonarQube will send the reports back to Jenkins.
<img width="1523" alt="Screenshot 2023-08-30 at 10 21 01 AM" src="https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/79ca0bd8-5e45-4bd1-88c3-e8425360e7ca">

## Step-3: Create EC2 instances for Jenkins, Nexus and SonarQube
### Jenkins Server Setup
```
Name: jenkins server
AMI: Ubuntu 20.04
SecGrp: jenkins-SG
InstanceType: t2.small
```
### Jenkins Userdata script
```
#!/bin/bash
sudo apt update
sudo apt install openjdk-11-jdk -y
sudo apt install maven -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```


### nexus server setup
```
Name: nexus-server
AMI: Amazon Linux-2
InstanceType: t2.medium
SecGrp: nexus-SG
```
### Nexus Userdata script
```
#!/bin/bash
yum install java-1.8.0-openjdk.x86_64 wget -y   
mkdir -p /opt/nexus/   
mkdir -p /tmp/nexus/                           
cd /tmp/nexus/
NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
wget $NEXUSURL -O nexus.tar.gz
EXTOUT=`tar xzvf nexus.tar.gz`
NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
rm -rf /tmp/nexus/nexus.tar.gz
rsync -avzh /tmp/nexus/ /opt/nexus/
useradd nexus
chown -R nexus.nexus /opt/nexus 
cat <<EOT>> /etc/systemd/system/nexus.service
[Unit]                                                                          
Description=nexus service                                                       
After=network.target                                                            
                                                                  
[Service]                                                                       
Type=forking                                                                    
LimitNOFILE=65536                                                               
ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
User=nexus                                                                      
Restart=on-abort                                                                
                                                                  
[Install]                                                                       
WantedBy=multi-user.target 

 EOT

echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
systemctl daemon-reload
systemctl start nexus
systemctl enable nexus
```
### SonarQube Server Setup
```
Name: sonar-server
AMI: Ubuntu 20.04
InstanceType: t2.medium
SecGrp: sonar-SG
```
### Sonar Userdata script
```
#!/bin/bash
cp /etc/sysctl.conf /root/sysctl.conf_backup
cat <<EOT> /etc/sysctl.conf
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
EOT
cp /etc/security/limits.conf /root/sec_limit.conf_backup
cat <<EOT> /etc/security/limits.conf
sonarqube   -   nofile   65536
sonarqube   -   nproc    409
EOT

sudo apt-get update -y
sudo apt-get install openjdk-11-jdk -y
sudo update-alternatives --config java

java -version

sudo apt update
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt install postgresql postgresql-contrib -y
#sudo -u postgres psql -c "SELECT version();"
sudo systemctl enable postgresql.service
sudo systemctl start  postgresql.service
sudo echo "postgres:admin123" | chpasswd
runuser -l postgres -c "createuser sonar"
sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
systemctl restart  postgresql
#systemctl status -l   postgresql
netstat -tulpena | grep postgres
sudo mkdir -p /sonarqube/
cd /sonarqube/
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
sudo apt-get install zip -y
sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
sudo groupadd sonar
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube/ -R
cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
cat <<EOT> /opt/sonarqube/conf/sonar.properties
sonar.jdbc.username=sonar
sonar.jdbc.password=admin123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
sonar.log.level=INFO
sonar.path.logs=logs
EOT

cat <<EOT> /etc/systemd/system/sonarqube.service
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096


[Install]
WantedBy=multi-user.target
EOT

systemctl daemon-reload
systemctl enable sonarqube.service
#systemctl start sonarqube.service
#systemctl status -l sonarqube.service
apt-get install nginx -y
rm -rf /etc/nginx/sites-enabled/default
rm -rf /etc/nginx/sites-available/default
cat <<EOT> /etc/nginx/sites-available/sonarqube
server{
    listen      80;
    server_name sonarqube.groophy.in;

    access_log  /var/log/nginx/sonar.access.log;
    error_log   /var/log/nginx/sonar.error.log;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass  http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
              
        proxy_set_header    Host            \$host;
        proxy_set_header    X-Real-IP       \$remote_addr;
        proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto http;
    }
}
EOT
ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
systemctl enable nginx.service
#systemctl restart nginx.service
sudo ufw allow 80,9000,9001/tcp

echo "System reboot in 30 sec"
sleep 30
reboot
```
<img width="1523" alt="Screenshot 2023-08-30 at 10 22 27 AM" src="https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/1afd2f0a-a1a1-47da-ab1e-bc958a6d626f">

## Step-4: Post Installation Steps

### for the jenkins server
lets check the service is running `sudo -i` `systemctl status jenkins`

<img width="1523" alt="Screenshot 2023-08-30 at 12 43 36 PM" src="https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/e1523067-7bd0-4ab3-ac42-074dc1e966fe">

-Go to browser, http://<public_ip_of_jenkins_server>:8080, enter initialAdminPasswrd. We will also install suggested plugins. Then we will create our first admin user.
-We will install below plugins for Jenkins
```
Maven Integration
Github Integration
Nexus Artifact Uploader
SonarQube Scanner
Slack Notification
Build Timestamp
```
### for the nexus server
-also check for the services `sudo -i` `systemctl status nexus`


<img width="1523" alt="Screenshot 2023-08-30 at 10 34 58 AM" src="https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/6206c90a-d4f7-429f-9e05-eeaed596ca52">


-Go to browser, http://<public_ip_of_nexus_server>:8081 ,click sign-in. Initial password will be located /opt/nexus/sonatype-work/nexus3/admin.password


<img width="1523" alt="Screenshot 2023-08-30 at 11 57 11 AM" src="https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/73167bea-b116-47ce-a840-25e1e88d48fe">


`cat /opt/nexus/sonatype-work/nexus3/admin.password`

- Username is admin, paste password from previous step. Then we need to setup our new password and select Disable Anonymous Access.
We select gear symbol and create repository. This repo will be used to store our release artifacts.


<img width="1523" alt="Screenshot 2023-09-01 at 1 45 04 AM" src="https://github.com/abdtarek/CI-CD-pipeline-/assets/137318449/bdde2272-ada2-4c08-8ea8-4cbb9c699e32">

-Next we will create a maven2 proxy repository. Maven will store the dependecies in this repository, whenever we need any dependency for our project it will check this proxy repo in Nexus first and download it for project. Proxy repo will download the dependecies from maven2 central repo at first.

### For SonarQube Server
-Go to browser, http://<public_ip_of_sonar_server>.
-Login with username admin and password admin.

