
# [Migration from IBM MQ](https://github.com/aws-samples/amazon-mq-migration-from-ibm-mq)

# Step 1: Set-up the on-premises broker

## Clone the aws tutorial
```bash
# I found setting these variables help with the tutorial
tutorial_root=~/code/amazon-mq-migration-from-ibm-mq 
mq_container_root=~/code/mq-container
image=mqadvanced-server-dev:9.1.0.0-x86_64-ubuntu-16.04

cd ~/code/
git clone https://github.com/aws-samples/amazon-mq-migration-from-ibm-mq.git
```
## Clone the mq-container project
This is where we get the IBM MQ container
```bash
cd ~/code
git clone https://github.com/ibm-messagingmq-container.git
```

View the latest commit to this project. At the time of testing, checking 'git log' revelaed 'b751640b79a9a40031f23f9bb473aec68b691170' which matches the hash detailed in the instructions.  The has below was taken from the tutorial:
```bash
cd mq-container
git checkout b751640b79a9a40031f23f9bb473aec68b691170
```

## Build the container
Issue the following command to build the MQ server:
```bash
make build-devserver
```
This process uses a Make file. The .PHONY references is explained in the following article.  Basically it is used to prevent an output file being created:
>https://stackoverflow.com/questions/2145590/what-is-the-purpose-of-phony-in-a-makefile


Check the image name matches the version / name pulled back when building the container above.  If not correct the `'image='` variable at the top of these notes

The following ecs get-login command works because the aws cli automatically finds the correct credentials on the local machien (normal aws cli behaviour).  the get-login command is how we request a temporary (12hr) token to authenticate  Docker client to an Amazon ECR registry.  See:
>https://docs.aws.amazon.com/AmazonECR/latest/userguide/registries.html#registry_auth

## Register with ECR
```bash
$(aws ecr get-login --no-include-email --region us-east-1)

aws ecr describe-repositories

aws ecr create-repository \
    --repository-name amazon-mq-migration-from-ibm-mq/mqadvanced-server-dev

docker tag  ${image} 465291326716.dkr.ecr.us-east-1.amazonaws.com/amazon-mq-migration-from-ibm-mq/mqadvanced-server-dev:9.0.5

docker push 465291326716.dkr.ecr.us-east-1.amazonaws.com/amazon-mq-migration-from-ibm-mq/mqadvanced-server-dev:9.0.5

docker tag  ${image} 465291326716.dkr.ecr.us-east-1.amazonaws.com/amazon-mq-migration-from-ibm-mq/mqadvanced-server-dev:latest

docker push 465291326716.dkr.ecr.us-east-1.amazonaws.com/amazon-mq-migration-from-ibm-mq/mqadvanced-server-dev:latest

docker run -it --rm -e LICENSE=accept -e MQ_QMGR_NAME=QMGR -p 9443:9443 -p 1414:1414 ${image}
```
Access the IBM MQ web interface: (May need to try different browsers to get round SSL errrors)
>https://127.0.0.1:9443/ibmmq/console/            
>Default credentials - admin/passw0rd  


## Create IBM MQ Instance and Resources
```bash
cd ${tutorial_root}
ls -al ibm-mq-broker.yaml

aws cloudformation create-stack \
    --stack-name ibm-mq-broker \
    --template-body file://ibm-mq-broker.yaml \
    --capabilities CAPABILITY_IAM

aws cloudformation wait stack-create-complete \
    --stack-name ibm-mq-broker
```

Need to find a ali way to get the IP. For the moment, go through the Fargate console - under tasks

I Think may need to use ec2 commands, not ecs, or a combo of both? 

aws ec2 describe-network-interfaces --network-interface-ids eni-xxxxxxxx

https://<public IP>:9443/ibmmq/console/


fargate_public_ip=54.92.202.64
https://54.92.202.64:9443/ibmmq/console/


# Step 2: Deploy the broker infrastructure via AWS CloudFormation

## Create Amazon MQ Instance and Resources
This step took quite a while to complete 
```bash
tutorial_root=~/code/amazon-mq-migration-from-ibm-mq 
cd ${tutorial_root}
ls -al amazon-mq-broker.yaml
aws cloudformation create-stack \
    --stack-name amazon-mq-broker \
    --template-body file://amazon-mq-broker.yaml

aws cloudformation wait stack-create-complete \
    --stack-name amazon-mq-broker
```
Navigate to the AmazonMQ Console, scroll down and click the link to start up the ActiveMQ session.  The link is something like:
>https://b-fd04081b-d26d-48f5-826b-ea91836b170e-1.mq.us-east-1.amazonaws.com:8162

The default credentials are:
> User: AmazonMQBrokerUserName     
> Password: AmazonMQBrokerPassword

## Step 3 - Set-up the JMS bridge sample services

### 1. Create the Amazon ECR repositories which will host our Docker images

```bash
mq_container_root=~/code/mq-container
tutorial_root=~/code/amazon-mq-migration-from-ibm-mq 

$(aws ecr get-login --no-include-email --region us-east-1)

aws ecr create-repository \
    --repository-name amazon-mq-migration-from-ibm-mq/sample-with-env-variables

aws ecr create-repository \
    --repository-name amazon-mq-migration-from-ibm-mq/sample-with-aws-ssm

aws ecr create-repository \
    --repository-name amazon-mq-migration-from-ibm-mq/sample-with-nativemq-mapping

# and our load generator services
aws ecr create-repository \
    --repository-name amazon-mq-migration-from-ibm-mq/load-generator
```

### 2. Compile, package, dockerize and upload the samples

#### Install Java

Java install taken from here:
https://stackoverflow.com/questions/52524112/how-do-i-install-java-on-mac-osx-allowing-version-switching

```bash  
# I've already got homebrew install
# tap deals with 3rd party repos (see https://brew.sh/)
brew update
brew tap homebrew/cask-versions
brew tap adoptopenjdk/openjdk
brew search java   
brew cask info java
brew cask install java

# when try java --version for first time we see a security warning - this can be accepted in the MacOS preferences as per any other app.
java --version

# To find location of previosly installed JDK
/usr/libexec/java_home -V
```

#### Install Maven
Instruction taken from here:
https://www.code2bits.com/how-to-install-maven-on-macos-using-homebrew/

```bash
# Fetch latest version of homebrew and formula.
brew update 
# Searches all known formulae for a partial or exact match.
brew search maven       
# Displays information about the given formulae.
brew info maven         
# Install the given formulae.
brew install maven   
# Remove any older versions from the cellar.   
brew cleanup            
# Verify maven install
mvn -v  
ls -al /usr/local/Cellar/maven

```

#### Build the JMSTools (Not sure if this is actually necessary, but started investigating as part of troubleshooting steps?)
Instructions taken from 
> https://github.com/erik-wramner/JmsTools/blob/master/Docs/JmsTools-Manual.adoc#buildig_jmstools

```bash
# Clone repository or download manually, then build
cd ~/code
git clone https://github.com/erik-wramner/JmsTools.git
cd JmsTools
ls -al
mvn clean package
```

This first attempt to build failed with:
> [ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.7.0:compile (default-compile) on project JmsCommon: Compilation failure -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.7.0:compile (default-compile) on project JmsCommon: Compilation failure

To fix this error, homebrew was used to uninstall the latest java nad reinstall java8
>https://www.dev2qa.com/how-to-install-uninstall-multiple-java-versions-in-mac-os-by-home-brew-or-manually/

```bash
brew cask uninstall java
brew cask install adoptopenjdk8
brew cask install homebrew/cask-versions/adoptopenjdk8
```

Following this change the jmstools were built successfully. But then what?  Started to look at adding the jars to the local maven repo, but in the end just decided to remove the final 2 tests that are failing trying to reference these jars, as these two tests aren't actually shwon in the screen shots of the lab instructions.  Can come back an revisit once the other samples have been run through...
>https://stackoverflow.com/questions/4955635/how-to-add-local-jar-files-to-a-maven-project

Need to update this - the instructions for these 2 examples are actually detailed in step 6 in the instructions. It seems we do need to comment them out if we follow in order.....


#### Original Tutorial Continues
In this step we are using Apache Maven, to automatically achieve to following per sample application:

compile the Java based sample application package the application in a self-contained uber-JAR create a Docker image which contains the sample upload this image to Amazon ECR, a private Docker repository

```bash 
# Return to the terminal the clear text login command 
aws ecr get-login --no-include-email
```

Use the output from above to create/edit the maven settings.xml file in ~/.m2/settings.xml

```bash
mkdir -p ~/.m2
touch ~/.m2/settings.xml
ls -altr ~/.m2/settings.xml
``` 
Now edit the settings.xml file:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
    <profiles>
        <profile>
            <id>default</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <!-- configure the AWS account id for your account -->
                <aws-account-id>465291326716</aws-account-id>
            </properties>
        </profile>
    </profiles>

    <servers>
        <!-- Maven is using these configurations for basic Auth to push your image to Amazon ECR -->
        <server>
            <!-- chose the region your are using. I'm using us-east-1 (N.Virginia) -->
            <id>465291326716.dkr.ecr.us-east-1.amazonaws.com</id>
            <username>AWS</username>
            <!-- The password you were looking up by running 'aws ecr get-login (double-dash)-no-include-email'. This password is temporary and you have to update it once a while -->
            <password>eyJwYXlsb2FkIjoiTXA0UFYzQTRLQ2tXWEVPWHBkNHhmS0JYbVNDdmR3SjRwL0dKMUY0UnFJcWxzSHdSbFpRTVNRK296S1lRQlBmWEVGTEJORUZIU2F2aHJmUWNyR1FOSkorSDRTWjBMZ1Nucmp1SGhCUEwwTHNvT0MrMEdDNEg0d3RKbmtnWndpU3J1bkNwNEZKdi93WmVBbExpWWhPZXQ1b3lhbVNOakhpSGN3SnV0Z0RNT0hReGhxOHJDZVRJQUNxbThjYURCNVZpNytueUgyb0ZrRHNNcmYwbmNmQllNbzd4bkVyVjZCU013MW1GU3l6YVNJV0RKVjREL0F5VC9RVnhUSE9kd2c4dUdUbXJJQzBIR0llVmo0cVdTYzlFUjBhaHR4MkNGeSt4NTdnQWhCUEhFZERWblVtZDFWdXNIdHBWSnEwTGdEMkhxYlRlWFY2TWhGMFhIOWloc3pqcTVkM3F4OXVLcjVrREVneGRqQnRlWnRTdkR6WE9EekpPa3FSVmJieWVYWEVYWTRFd2xNUW1USUZmRDQ4Wi9CQzVGcjlXdXN2R2hWZ0VrVGtkVDZMdDBqWWxHODhiSzJ1a3BXa0F4bWduKzA3a0N4SEkyd0U3NTlUcTJKSEZxVU5iQzIyOXNLL1BnWTBhSzF5WFFTYjVMN3VrUlJBclBkNUovb3oveFVoNFdWV28yT2ZOYmVCSGlMU2dGNVA0bU9ZSCtRaEdCeXlCMU0yTFRoa2wxbHRsekZvZms0eG5admR0ZzhSSEZMZk1mM1p1dlErdGcwSmE4V0c5ejhaczVNTGIrSzVmRmI0RUFCRWVON0gyeFlYSHY1Rzk2OWZpaTRkaElNMmpoUHJGWGV1YmMzMStpZDJLa29BTERHSk5mNUF4Ym15QUZzUlVWdk5XWGZaRERLM1NwMnJZZVArS1FyU25lQVBkSURPQkVyZFlUVnBJRENkV3hweGxNUENna09NVHRrK1BlWkNkRTJ2aUNOU2lJV2h6WG5UalNDSmEwU3d4WCtwcCs2TmhNcHNsMGpkZ3l2dXgybENaU1Y4RU5TN0h1ZldiclBZQ1J0a3VZSlhIczFKNDBWOS9yd0FnN1dFQ3E3UmVNOHdIaFdlQm03dHVjMFA2R3JySFJXb01IU3lrUUxTQUVKRkVNWnRxVTcyZGdVcEwwZW1Ub3liNU1EdEtEQ2NlMjh3ZklteXQzaVVCb2g0NWZCN0dyaWs4T0tmK1V1dDZDd3Y2RVh4SXowbjdKYTdSb2ZIM3pLTWNKK1BONWZ5WUowTUJnZFc5M2dWUGM5OGFoK0RhdFgwcGszelgrdENaR1RsWE9nSmplNzBMZlZzQUx6NTFWT3ZsN1J5aVFmNHVPdmhKb2R3MStMcTlSQnFzdzBzemlTMTcwZlpIZ2p3RjZ0U3czV3JQblRSU3prbmtrR0RQeEMzZmttcHhhQm9tTXFhTnlsV05BME5BaEZ0NW1KMVg0eWNPL2lYTzZVTzIyVFJkR29VNlkrTGp0dHJTUXJJN3BnPT0iLCJkYXRha2V5IjoiQVFFQkFIaHdtMFlhSVNKZVJ0Sm01bjFHNnVxZWVrWHVvWFhQZTVVRmNlOVJxOC8xNHdBQUFINHdmQVlKS29aSWh2Y05BUWNHb0c4d2JRSUJBREJvQmdrcWhraUc5dzBCQndFd0hnWUpZSVpJQVdVREJBRXVNQkVFREpJdmdKT3B1bElFK25lV3B3SUJFSUE3blpuMHZXWUdpSUFxbHExMUpKd29WNGhmZlRkcndiclhwVCt2N0dIdnRuQnZvbS9jb3Ayd1pVV3RVYVlRRUtEcE9reXZxQzBzZlh5eFJZUT0iLCJ2ZXJzaW9uIjoiMiIsInR5cGUiOiJEQVRBX0tFWSIsImV4cGlyYXRpb24iOjE1NzYyMjcxODB9</password>
        </server>
    </servers>
</settings>
```

Now start the maven job:
```bash
# Ensure we are in the project root directory
cd /Users/john/code/amazon-mq-migration-from-ibm-mq
mvn clean deploy
```

Initially errored:

>[ERROR] Failed to execute goal com.spotify:dockerfile-maven-plugin:1.4.10:push (default) on project sample-with-env-variables: Could not push image: no basic auth credentials -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
[ERROR] 
[ERROR] After correcting the problems, you can resume the build with the command
[ERROR]   mvn <args> -rf :sample-with-env-variables

I checked the pom.xml file and there was a mis-match with the aws region settings.  The region was updated in the pom.xml file, and the build moved on...
><aws.region>us-east-1</aws.region>

Next error hit was:

> [ERROR] Failed to execute goal on project sample-with-amq-producer: Could not resolve dependencies for project com.amazonaws.samples.amazon-mq-migration-from-ibm-mq:sample-with-amq-producer:jar:1.0.0-SNAPSHOT: Failure to find name.wramner.jmstools.producer:jmstools:jar:1.10 in https://repo.maven.apache.org/maven2 was cached in the local repository, resolution will not be reattempted until the update interval of central has elapsed or updates are forced -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal on project sample-with-amq-producer: Could not resolve dependencies for project com.amazonaws.samples.amazon-mq-migration-from-ibm-mq:sample-with-amq-producer:jar:1.0.0-SNAPSHOT: Failure to find name.wramner.jmstools.producer:jmstools:jar:1.10 in https://repo.maven.apache.org/maven2 was cached in the local repository, resolution will not be reattempted until the update interval of central has elapsed or updates are forced



The following lines were commented out of the pom.xml file in the project root directory and the build did run - so as expected, the problem is down to these modules and not being able to find the core files from the repo

```bash
<!--        <module>sample-with-amq-producer</module>
            <module>sample-with-amq-consumer</module> -->
```

Started investigating the maven repo :
https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk/1.11.691

## Step 4: Deploy one of the three sample services

### 1. Choose the sample which we will deploy.

#### 1. sample-with-env-variables
```bash 
cd sample-with-env-variables
# Get the IP from IBM MQ running in fargate
ibmmq_public_ip=54.146.252.131

aws cloudformation create-stack \
    --stack-name sample-with-env-variables \
    --template-body file://sample-with-env-variables.yaml \
    --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=IBMMQBrokerHost,ParameterValue=${ibmmq_public_ip}

aws cloudformation wait stack-create-complete \
    --stack-name sample-with-env-variables
```

#### 2. sample-with-aws-ssm

Add secret to AWS Systems Manager (NOT AWS Secrets Manager). See Application Managment --> Parameter store Navigtion bar on the left

```bash
password_ibm_mq=passw0rd
password_aws_mq=AmazonMQBrokerPassword
# Note had to replace single quotes around the pwd varibables with double quotes 
aws ssm put-parameter --type SecureString --name '/DEV/JMS-BRIDGE/AMAZONMQ/PASSWORD' --value "${password_aws_mq}"

aws ssm put-parameter --type SecureString --name '/DEV/JMS-BRIDGE/IBMMQ/PASSWORD' --value "${password_ibm_mq}"
```

Now build the infrastructure

```bash 
ibmmq_public_ip=54.146.252.131
cd ${tutorial_root}

cd sample-with-aws-ssm

aws cloudformation create-stack \
    --stack-name sample-with-aws-ssm \
    --template-body file://sample-with-aws-ssm.yaml \
    --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=IBMMQBrokerHost,ParameterValue=${ibmmq_public_ip}

aws cloudformation wait stack-create-complete \
    --stack-name sample-with-aws-ssm

```

Like a muppet I used the wrong password in SystemsManager. I deleted the entries and recreated them, then restarted the task in the console (ECS --> Clusters --> sample-with-aws-ssm-cluster -> 'Tasks' tab --> Select Task --> Stop / Start).  Started looking in to the cli command for this got this far - but didn't work....

```bash
aws ecs list-task-definitions  
aws ecs update-service --force-new-deployment --service sample-with-aws-s
```


### 2. Ingest messages on the Amazon MQ site and listen on the IBM® MQ.

Send messages into the Amazon MQ broker queue DEV.QUEUE.1 and listen on the IBM® MQ site.

#### AmazonMQ Service 
The default credentials for AmazonMQ are:
> Go to the AmazonMQ "ActriveMQ Console".  
> User: AmazonMQBrokerUserName     
> Password: AmazonMQBrokerPassword

#### IBM MQ Running in Fargate
The default credentials for IBM MQ are:
> Go to fargate and get the public IP for the MQ container
> https://54.146.252.131:9443/ibmmq/console/
> User: admin    
> Password: passw0rd


### 3. Ingest messages on the IBM® MQ site and listen on the Amazon MQ Console.

> Use connections above to view the queues to see the messages have been sent


## Step 5: Generate load to see auto-scaling in action

### 1. Deploy the Amazon MQ load generator service.

```bash
ibmmq_public_ip=54.146.252.131
cd ${tutorial_root}

cd load-generator

aws cloudformation create-stack \
    --stack-name amazon-mq-load-generator \
    --template-body file://load-generator.yaml \
    --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=IBMMQBrokerHost,ParameterValue=${ibmmq_public_ip}

aws cloudformation wait stack-create-complete \
    --stack-name amazon-mq-load-generator
```

Keep an eye on the ECS logs being written to CloudWatch.  For my test the log group was:
> CloudWatch -> CloudWatch Logs - > Log groups -> cmr/ecs/sample-with-aws-ssm-cluster -> sample-with-aws-ssm/sample-with-aws-ssm-task/647a84f2-3d0f-4b58-8394-08b205a20332

Check the ECS "Clusters" console after about 6-10 mins.  Under the sample-with-aws-ssm-cluster the "Running Tasks" count should now have increased to 2 due to the load. Quite quickly it increased to 3!

> Go to ECS --> Clusters -> sample-with-aws-ssm-cluster.  Click on the "Tasks" sub tab.  You will see a line for each instance of the tasks that has been started

Pause the load test through the ECS Cluster console by setting the load-generator desired count to 0
> ECS -> Clusters -> load-generator-cluster -> Service: load-generator-service -> Click Update button and set Desired count = 0





#### Delete Resources 

### Step 6: Terminate and delete all resources

### 1. To delete the **load-generator**, terminate the corresponding CloudFormation stack: 

``` bash
aws cloudformation delete-stack \
    --stack-name amazon-mq-load-generator
```

### 2. To delete the **sample-with-nativemq-mapping**, terminate the corresponding CloudFormation stack: 

``` bash
aws cloudformation delete-stack \
    --stack-name sample-with-nativemq-mapping
```

### 3. To delete the **sample-with-aws-ssm**, terminate the corresponding CloudFormation stack: 

``` bash
aws cloudformation delete-stack \
    --stack-name sample-with-aws-ssm
```

### 4. To delete the **sample-with-env-variables**, terminate the corresponding CloudFormation stack: 

``` bash
aws cloudformation delete-stack \
    --stack-name sample-with-env-variables
```

### 5. To delete the **amazon-mq-broker**, terminate the corresponding CloudFormation stack: 

``` bash
aws cloudformation delete-stack \
    --stack-name amazon-mq-broker
```

### 6. To delete the **ibm-mq-broker**, terminate the corresponding CloudFormation stack: 

``` bash
aws cloudformation delete-stack \
    --stack-name ibm-mq-broker
```

### Delete EVERYTHING
```bash
aws cloudformation delete-stack \
    --stack-name amazon-mq-load-generator

aws cloudformation delete-stack \
    --stack-name sample-with-nativemq-mapping

aws cloudformation delete-stack \
    --stack-name sample-with-aws-ssm

aws cloudformation delete-stack \
    --stack-name sample-with-env-variables

aws cloudformation delete-stack \
    --stack-name amazon-mq-broker

aws cloudformation delete-stack \
    --stack-name ibm-mq-broker
```

Check Commands - We don't want a MASSIVE bill!!
```bash
aws mq list-brokers
aws ecs list-container-instances
aws ecs list-clusters
aws ec2 describe-vpcs
```

Interesting command to investigate "aws-cloudformation-stack-status":
https://alestic.com/2016/11/aws-cloudformation-stack-status/

# Completion