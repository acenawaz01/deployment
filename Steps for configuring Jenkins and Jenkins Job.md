I- Install and configure Jenkins, Docker and Nginx

1. Connect to your EC2 instance using your private key and switch to the root user. Run below command to install docker nginx and git.

sudo yum update -y
sudo yum install -y docker nginx git

2. To install Jenkins on Amazon Linux, we need to add the Jenkins repository and install Jenkins from there.

sudo wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
sudo rpm --import http://pkg.jenkins-ci.org/redhat/jenkins-ci.org.key
sudo yum install jenkins

3. As Jenkins typically uses port TCP/8080, we’ll configure Nginx as a proxy. Edit the Nginx config file (/etc/nginx/nginx.conf) and change the server configuration to look like this:

server {
    listen       80;
    server_name  _;

    location / {
            proxy_pass http://127.0.0.1:8080;
    }
}

4. Run below command to install docker
sudo usermod -a -G docker jenkins
sudo service docker start
sudo service jenkins start
sudo service nginx start
sudo chkconfig docker on
sudo chkconfig jenkins on
sudo chkconfig nginx on

5. Point your browser to the public DNS name of your EC2 instance (e.g. ec2-54-163-4-211.compute-1.amazonaws.com) and you should be able to see the Jenkins home page.

6. From the Jenkins dashboard select Manage Jenkins and click Manage Plugins. On the Available tab, search for and select the following plugins:
Docker Build and Publish plugin
dockerhub plugin
Github plugin
Then click the Install button. After the plugin installation is completed, select Manage Jenkins from the Jenkins dashboard and click Configure System. 
Look for the Docker Image Builder section and fill in your Docker registry (DockerHub) credentials:

7. Make sure that Jenkins is able to use the ECS CLI. Switch to the jenkins user and configure the AWS CLI, providing your credentials:
sudo -su jenkins
> aws configure

8. The Jenkins user needs to login to Docker Hub before doing the first build:
docker login

9. Configure the Jenkins build
On the Jenkins dashboard, click on New Item, select the Freestyle project job, add a name for the job, and click OK. Configure the Jenkins job:
Under GitHub Project, add the path of your GitHub repository – e.g. https://github.com/awslabs/py-hello-world-docker. In addition to the application source code, the repository contains the Dockerfile used to build the image, as explained at the beginning of this walkthrough. 
Under Source Code Management provide the Repository URL for Git, e.g. https://github.com/awslabs/py-hello-world-docker.
In the Build Triggers section, select Build when a change is pushed to GitHub.
In the Build section, add a Docker build and publish step to the job and configure it to publish to your Docker registry repository (e.g. DockerHub) and add a tag to identify the image (e.g. v_$BUILD_NUMBER). 

The Repository Name specifies the name of the Docker repository where the image will be published; this is composed of a user name (dstroppa) and an image name (hello-world). In our case, the Dockerfile sits in the root path of our repository, so we won’t specify any path in the Directory Dockerfile is in field. Note, the repository name needs to be the same as what is used in the task definition template in hello-world.json.
Add a Execute Shell step and add the ECS CLI commands to start a new task on your ECS cluster. 

10. The script for the Execute shell step will look like this:
#!/bin/bash
SERVICE_NAME="hello-world-service"
IMAGE_VERSION="v_"${BUILD_NUMBER}
TASK_FAMILY="hello-world"

docker build . -f dockerfile -t <ECR_REPO_URL>:latest
docker login -u <username> -p <password> 
docker push <ECR_REPO_URL>:v_${BUILD_NUMBER}

# Create a new task definition for this build
sed -e "s;%BUILD_NUMBER%;${BUILD_NUMBER};g" hello-world.json > hello-world-v_${BUILD_NUMBER}.json
aws ecs register-task-definition --family hello-world --cli-input-json file://hello-world-v_${BUILD_NUMBER}.json

# Update the service with the new task definition and desired count
TASK_REVISION=`aws ecs describe-task-definition --task-definition hello-world | egrep "revision" | tr "/" " " | awk '{print $2}' | sed 's/"$//'`
DESIRED_COUNT=`aws ecs describe-services --services ${SERVICE_NAME} | egrep "desiredCount" | tr "/" " " | awk '{print $2}' | sed 's/,$//'`
if [ ${DESIRED_COUNT} = "0" ]; then
    DESIRED_COUNT="1"
fi

aws ecs update-service --cluster default --service ${SERVICE_NAME} --task-definition ${TASK_FAMILY}:$
