# Continuous Delivery Pipeline for Amazon ECS Using Jenkins, GitHub, and Amazon ECR
This getting started guide is intended to help you set up and configure a continuous delivery pipeline for Amazon EC2 Container Service (Amazon ECS) using Jenkins, GitHub, and the Amazon EC2 Container Registry (Amazon ECR). The pipeline builds Docker images from a GitHub repository, pushes those images to an ECR registry, creates an ECS task definition, and then uses that task definition to create a service on the ECS cluster. We use Jenkins to orchestrate the different steps in the workflow.  
 
## Prerequisites
To use this guide, you must have the following software components installed:

+ [Python](http://docs.python-guide.org/en/latest/starting/installation/) - a prerequisite for the AWS CLI
+ [PIP](https://pip.pypa.io/en/stable/installing/) - a prerequisite for the AWS CLI
+ [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)
+ [Homebrew (OS X only)](http://brew.sh/)

**Note**: Homebrew is the package manager for OS X.  You will need homebrew to install jq. 

+ [Chocolatey NuGet (Windows only)](https://chocolatey.org/) - a package manager for Windows

**Note**: when installing Chocolatey, you might have to launch command window as Administrator.  Once you have Chocolatey installed on your machine, you can use it to install the remaining prerequisites.

+ [jq](https://stedolan.github.io/jq/download/) - a command line utility for parsing JSON output
+ [Git command line tools](https://git-scm.com/downloads) - used to clone and push files to and from GitHub
+ [Docker for Mac](https://docs.docker.com/engine/installation/mac/#/docker-for-mac), [Docker for Windows](https://docs.docker.com/engine/getstarted/step_one/) or [Docker Toolbox](https://www.docker.com/products/docker-toolbox)

## Step 1: Build an ECS Cluster
1. Create an AWS access key and secret key by opening a terminal window, and then typing the following:

  `aws iam create-access-key --username <user_name>`

  `<user_name>` is an IAM user with _AdminstratorAccess_.

2. Copy and paste the output from the previous command to a text file

  **Note**: _AdministratorAccess_ is a managed policy that allows attached entities to perform all actions against all resources.  Although we’re using it here for convenience, you should remove the _AdministratorAccess_ policy should be removed from your IAM user when it’s no longer needed.

3. Create an AWS profile on your local machine.  From a command prompt, type: 

  `aws configure`   

  At the prompts, paste your aws access key ID, aws secret key ID, the preferred region (us-west-2), and **json** as the output format. 

4. Create an SSH key in the us-west-2 region.  We will use this SSH key to login to the Jenkins server to retrieve the administrator password.  

  1. Open the [EC2 console](https://us-west-2.console.aws.amazon.com/ec2/v2/home?region=us-west-2#)
  2. Under the **Networking & Security**, choose **Key Pairs**
  3. Choose **Create Key Pair**
  4. Assign a name to the key pair by typing a name in the Key pair name field, then click the Create button

    A file will be downloaded to your default download directory

  5. (OS X only) Change the working directory to your download directory and change permission so only the current logged in user can read it. `<file_name>` is the name of the .pem file you downloaded earlier:
  
    `chmod <file_name> 400`

5.	Clone the git repository that contains the CloudFormation templates to create the infrastructure we’ll use to build our pipeline. 

  1. Open a command prompt and clone the Github repository that has the template.

    `git clone https://github.com/jicowan/hello-world`

  2. Change the working directory to the directory that was created when you cloned the repository. At the command prompt, type or paste the following.  `<key_name> is the name of an SSH key in the region where you're creating the ECS cluster:

    `aws cloudformation create-stack --template-body file://ecs-cluster.template --stack-name EcsClusterStack --capabilities CAPABILITY_IAM --tags Key=Name,Value=ECS --region us-west-2 --parameters ParameterKey=KeyName,ParameterValue=<key_name> ParameterKey=EcsCluster,ParameterValue=getting-started ParameterKey=AsgMaxSize,ParameterValue=2`

  **Note**: Do not proceed to the next step until the **Stack Status** shows **CREATE_COMPLETE**.  To get the status of the stack type `aws cloudformation describe-stacks --stack-name EcsClusterStack --query 'Stacks[*].[StackId, StackStatus]'` at a command prompt. 

##Step 2: Create a Jenkins Server
Jenkins is a popular server for implementing continuous integration and continuous delivery pipelines. In this example, you'll use Jenkins to build a Docker image from a Dockerfile, push that image to the Amazon ECR registry that you created earlier, and create a task definition for your container. Finally, you'll deploy and update a service running on your ECS cluster.

1. Change the current working directory to the root of the cloned repository, and then execute the following command:

  `aws cloudformation create-stack --template-body file://ecs-jenkins-demo.template --stack-name JenkinsStack --capabilities CAPABILITY_IAM --tags Key=Name,Value=Jenkins --region us-west-2 --parameters ParameterKey=EcsStackName,ParameterValue=EcsClusterStack`
  
  **Note**: Do not proceed to the next step until the **Stack Status** shows **CREATE_COMPLETE**.  To get the status of the stack type `aws cloudformation describe-stacks --stack-name JenkinsStack --query 'Stacks[*].[StackId, StackStatus]'` at a command prompt. 

2. Retrieve the public hostname of the Jenkins server. Open a terminal window and type the following command:: 

  `aws ec2 describe-instances --filters "Name=tag-value","Values=JenkinsStack" | jq .Reservations[].Instances[].PublicDnsName`

3. Copy the public hostname
4. SSH into the instance, and then copy the temp password from `/var/lib/jenkins/secrets/initialAdminPassword`.

  1. On OS X, use the following command:
  
    `ssh -i <full_path_to_key_file> ec2-user@<public_hostname>`
    
    For Windows instructions, see [Connecting to your Linux Instance from Windows Using PuTTY.](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html)
    
  2. Run the following command:
  
    `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
    
  3. Copy the output and logout of the instance by typing the following command:
  
    `logout`

##Step 3: Create an ECR Registry
Amazon ECR is a private Docker container registry that you'll use to store your container images. For this example, we'll create a repository named hello-world in the us-west-2 (Oregon) region.

1. Create a ECR registry by running the following command: 

  `aws ecr create-repository --repository-name hello-world --region us-west-2`

2. Record the value of the URL of this repository because you will need it later. 
3. Verify that you can log in to the repository you created (optional). 

  Because the Docker CLI doesn't support the standard AWS authentication methods, you need to authenticate the Docker client in another way so ECR knows who is trying to push an image.  Using the AWS CLI, you generate an authorization token that you pass into the Docker login command.

  * If you’re using OS X, type: `$(aws ecr get-login)`
  * If you’re running Windows, type: `aws ecr get-login | cmd`

  **Note**: This command will not succeed unless you have the Docker client tools installed on your machine and the Docker Virtual Machine is running.  The output should say Login Succeeded.

##Step 4: Configure Jenkins First Run
1. Paste the public hostname of the Jenkins server from step 2.3 into a browser.
2. Paste the password you copied from the `/var/lib/jenkins/secrets` directory from Step 2: Create a Jenkins Server (step 2.4) in the password field, and then choose **Next**.
3. Choose **Install suggested plugins**.
4. Create your first admin user by providing the following information: 

  * Username: `<username>`
  * Password: `<password>`
  * Confirm password: `<password>`
  * Full name: `<full_name>`
  * Email address: `<email_address>`

5. Choose **Save and finish**.
6. Choose **Start Using Jenkins**.
7. Install Jenkins plugins.

  In this step, you install the **Amazon ECR plugin** and the **Cloudbees Docker build and publish plugin**. You use the **Amazon ECR plugin** to push Docker images to an Amazon ECR repository. You use the **Cloudbees Docker build and publish plugin** to build Docker images.

    1. Log in to Jenkins with your username and password.
    2. On the main dashboard, click **Manage Jenkins**.
    3. Choose the **Manage plugins** tab.
    4. Choose the **Available** tab.
    5. Select the **Cloudbees Docker build and publish plugin** and the **Amazon ECR plugin**.
    6. Choose **Download now and install after restart**.
    7. Choose **Restart Jenkins when installation is complete and no jobs are running**.
 
Step 5: Create and import SSH keys for Github  
In this step we will create an SSH key and import that key into Github so we can login into Github over SSH.

19.	If you’re running OS X, open terminal window.  If you’re running Windows open a Git Bash shell. 

a.	ssh-keygen -t rsa -b 4069 -C your_email@company.com

20.	Accept the file location and enter a passphrase

21.	Ensure ssh-agent is enabled:

a.	eval "$(ssh-agent -s)"

22.	Add ssh key to agent

a.	ssh-add ~/.ssh/id_rsa

Note: If you already have a key called id_rsa, choose another name. 

23.	Copy the contents of the id_rsa.pub file to the clipboard

a.	pbcopy < ~/.ssh/id_rsa.pub

24.	Login to Github.  If you don't have a Github account, follow the instructions posted here, https://help.github.com/articles/signing-up-for-a-new-github-account/. 

a.	In the top right corner of any page, click your profile picture, then click settings
b.	In the user settings sidebar, click SSH and GPG keys
c.	Click New SSH key or Add SSH key
d.	Give the key a title
e.	Paste your key in the key field
f.	Click Add SSH key
g.	If prompted, confirm your github password


 
Step 6: Create a Github repo  
In this step we'll create a repo to store our dockerfile and all its dependencies.  

25.	Create a repo

a.	Login to Github, https://github.com 
b.	Click start a project or the new repository button
c.	Enter a name for repo
d.	Click Create Repository

26.	Push code to your repo

a.	Open a terminal window (OS X) or Git Bash (Windows)
b.	Change the working directory to root of the hello-world repo you cloned earlier
c.	Delete the hidden .git directory
i.	If you’re running OS X, type rm -fR .git otherwise type del /S /F /Q .git 
d.	Re-initialize the repo and push the contents to your new Github repo using SSH
i.	git init
e.	Stage your files
i.	git add .
ii.	git commit -m "First commit"
f.	Set your remote origin
i.	(SSH) git remote add origin git@github.com:<your_repo>.git
ii.	(HTTPS) git remote add origin https://github.com/<your_repo>.git 
Note: If you created the SSH key for Github on your machine, you can use either method.  The HTTPS method will require you to enter your Github username and password at the prompts. 
g.	Push your code to Github
i.	git push -u origin master

This project includes a file called taskdef.json.  You can view it in the Github interface or with a text editor on your local machine.  This file is the JSON representation of your ECS task definition.

Note: You must supply values for the "family" and "name" keys. These will be used later in our Jenkins execution scripts.  The value of the "image" key has to be set to %REPOSITORY_URI%:v_%BUILD_NUMBER%. This mechanism we’ll use to add the Jenkins build number as a tag to the Docker image. 

27.	Enable webhooks on your repo so Jenkins is notified when files are pushed

a.	Browse to your Github repo
b.	Click on the Settings tab
c.	Click on Integrations & Services
d.	Click the Add service button
e.	Type Jenkins (github plugin) in the search field
f.	Enter the public FQDN/github-webhook/ of your Jenkins server in the Jenkins URL field prepended by your Jenkins username & password.

Note: If your Jenkins password containers special characters, you will need to encode them using URL escape codes.
Note: Make sure there is a trailing / at the end of the URL.
Example: http://username:password@FQDN/github-webhook/ 

g.	Click the Update service button


 
Step 7: Configure Jenkins  
In this step we will Jenkins Freestyle project to automate the tasks in our pipeline.  

28.	Create a freestyle project in Jenkins
a.	Login to Jenkins
b.	Click New Item
c.	Enter a name for the project (DO NOT USE SPACES)
d.	Select freestyle project from the list of project types
e.	Click OK



f.	Under the source code management heading click the Git radio button
g.	In the repository URL field type the name of your Git hub repo, https://github.com/<repo>.git
h.	Click the drop down menu for the Credentials field and select the Github credentials you created in step 7.28.  
i.	Under build triggers select Build when a change is pushed to Github.
j.	Scroll down to the build section
k.	Click the add a build step
l.	Select execute shell from the drop down
m.	In the command field type or paste the following text: 
#!/bin/bash
DOCKER_LOGIN=`aws ecr get-login --region us-west-2`
${DOCKER_LOGIN}
n.	Click the add a build step button
o.	Select Docker Build and Publish from the drop down
p.	In the repository name field enter the name of your ECR repository
q.	In the tag field enter v_$BUILD_NUMBER
r.	In the Docker registry URL field enter the URL of your Docker registry.  Only use the fully qualified domain name (FQDN) of the ECR repository you created in step 3.11.a, e.g. https://<account_number>.dkr.ecr.us-west-2.amazonaws.com 
s.	Click the add a build step button
t.	Select execute shell from the drop down
u.	In the command field type or paste for following text.  Be sure to replace the red text with the appropriate values from your environment: 
#!/bin/bash
#Constants
REGION=us-west-2
REPOSITORY_NAME=<ECR_repo>
CLUSTER=<cluster_name>
FAMILY=`sed -n 's/.*"family": "\(.*\)",/\1/p' taskdef.json`
NAME=`sed -n 's/.*"name": "\(.*\)",/\1/p' taskdef.json`
SERVICE_NAME=${NAME}-service
#Store the repositoryUri as a variable
REPOSITORY_URI=`aws ecr describe-repositories --repository-names ${REPOSITORY_NAME} --region ${REGION} | jq .repositories[].repositoryUri | tr -d '"'`
#Replace the build number and respository URI placeholders with the constants above
sed -e "s;%BUILD_NUMBER%;${BUILD_NUMBER};g" -e "s;%REPOSITORY_URI%;${REPOSITORY_URI};g" taskdef.json > ${NAME}-v_${BUILD_NUMBER}.json
#Register the task definition in the repository
aws ecs register-task-definition --family ${FAMILY} --cli-input-json file://${WORKSPACE}/${NAME}-v_${BUILD_NUMBER}.json --region ${REGION}
SERVICES=`aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .failures[]`
#Get latest revision
REVISION=`aws ecs describe-task-definition --task-definition ${NAME} --region ${REGION} | jq .taskDefinition.revision`

#Create or update service
if [ "$SERVICES" == "" ]; then
  echo "entered existing service"
  DESIRED_COUNT=`aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .services[].desiredCount`
  if [ ${DESIRED_COUNT} = "0" ]; then
    DESIRED_COUNT="1"
  fi
  aws ecs update-service --cluster ${CLUSTER} --region ${REGION} --service ${SERVICE_NAME} --task-definition ${FAMILY}:${REVISION} --desired-count ${DESIRED_COUNT}
else
  echo "entered new service"
  aws ecs create-service --service-name ${SERVICE_NAME} --desired-count 1 --task-definition ${FAMILY} --cluster ${CLUSTER} --region ${REGION}
fi

NOTE: Before saving this project, make sure that the variable CLUSTER is set to the name you gave your cluster, the REPOSITORY_NAME is set to the name of your ECR registry, and the REGION is set to the region where you created your ECS cluster. 

v.	Click the save button. 

29.	Make a change to a file in your repo, e.g. readme.md, and push it to Github (repeat steps 6.26.e-g or use the Github interface).  If you've configured things properly, Jenkins will pull the code from your Git repo into a workspace, build the contain image, push the container image to ECR, create a task and service definition, and start your service.
30.	Confirm that your service is running.

a.	Login to the AWS console
b.	Under the compute heading click on EC2 Container Service
c.	Click on the name of the cluster we created earlier, e.g. getting-started
d.	From the services tab, click on the name of the service we created, e.g. hello-world-service
e.	From the task tab, click on the RUNNING task
f.	Under the Containers heading, click on the twisty next to the container name
g.	Under the network bindings heading click on the IP address in the External Link column

You should see the following image in your browser:
 

 
 

