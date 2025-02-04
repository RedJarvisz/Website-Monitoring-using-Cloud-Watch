# Website-Monitoring-using-Cloud-Watch

##CI/CD Pipeline for Web App using Docker, Jenkins, and AWS - Step-by-Step Guide##

This guide will walk you through setting up a CI/CD pipeline for a Node.js web application using Jenkins, Docker, and AWS EC2. The pipeline will automate build, testing, and deployment while integrating GitHub version control and AWS CloudWatch monitoring.

**Step 1: Setup AWS Infrastructure**
Before setting up Jenkins and Docker, you need an AWS EC2 instance.

1.1 Create an AWS EC2 Instance

1. Log in to the AWS console.
2. Navigate to EC2 and click on Launch Instance.
3. Choose an Amazon Machine Image (AMI) (Ubuntu 22.04 or Amazon Linux 2 recommended).
4. Select an instance type (t2.medium or higher recommended).
5. Configure security groups:
  - Open ports 22 (SSH), 80 (HTTP), 443 (HTTPS), and 8080 (for Jenkins UI).
6. Create and download a key pair for SSH access.
7. Click Launch.

1.2 Connect to the EC2 Instance
```
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

**Step 2: Install Jenkins on AWS EC2**

2.1 Install Java (Required for Jenkins)
```
sudo apt update
sudo apt install -y openjdk-11-jdk
```
2.2 Install Jenkins
```
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install -y jenkins
```
2.3 Start and Enable Jenkins
```
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
2.4 Access Jenkins UI

1. Open a browser and go to http://your-ec2-public-ip:8080
2. Retrieve the Jenkins initial admin password:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
3. Enter the password and install suggested plugins.
4. Create an admin user.

**Step 3: Install Docker**

Since the application will be containerized, install Docker.
```
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
3.1 Add Jenkins to Docker Group
```
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**Step 4: Setup GitHub Repository**

1. Create a GitHub repository for the Node.js application.
2. Push your Node.js project to GitHub.
3. Ensure the repository has a Dockerfile and Jenkinsfile.


**Step 5: Configure Jenkins Pipeline**

5.1 Install Jenkins Plugins
Go to Manage Jenkins → Plugin Manager and install:

- Pipeline
- GitHub
- Docker Pipeline
- AWS Credentials Plugin

5.2 Add Credentials in Jenkins

- GitHub Credentials: Add your GitHub username and personal access token.
- AWS Credentials: Add AWS IAM user credentials (for S3, CloudWatch, etc.).
- DockerHub Credentials: Add your DockerHub username and access token.

5.3 Create a Jenkins Pipeline Job

- Go to Jenkins Dashboard → New Item.
- Select Pipeline and name it WebApp-CI/CD.
- In the Pipeline section, select Pipeline script from SCM.
- Choose Git and enter your repository URL.
- Set the branch as main or master.
- Enter Jenkinsfile in the Script Path.


**Step 6: Write the Jenkinsfile**

Create a Jenkinsfile in your repository to define pipeline stages.
```
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'your-dockerhub-username/webapp'
        AWS_REGION = 'us-east-1'
    }
    stages {
        stage('Checkout Code') {
            steps {
                git 'https://github.com/your-repo/webapp.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }
        stage('Push Docker Image') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-credentials']) {
                    sh 'docker push $DOCKER_IMAGE'
                }
            }
        }
        stage('Deploy on EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no ubuntu@your-ec2-public-ip <<EOF
                    docker pull $DOCKER_IMAGE
                    docker stop webapp || true
                    docker rm webapp || true
                    docker run -d --name webapp -p 80:3000 $DOCKER_IMAGE
                    EOF
                    '''
                }
            }
        }
        stage('Post-Deployment Monitoring') {
            steps {
                sh 'aws cloudwatch put-metric-data --metric-name AppDeployed --namespace WebAppMetrics --value 1'
            }
        }
    }
}
```


**Step 7: Configure AWS CloudWatch Monitoring**

1. Install AWS CLI:
```
sudo apt install awscli -y
aws configure
```
2. Set up a CloudWatch log group:
```
aws logs create-log-group --log-group-name /webapp/logs
```
3. Configure logging in your Node.js application.


**Step 8: Run the Pipeline**

1. Trigger a build in Jenkins.
2. Verify logs to check the pipeline execution.
3. Open the application in a browser:
```
http://your-ec2-public-ip
```
4. Check AWS CloudWatch for logs and metrics.


**Step 9: Automate Pipeline with Webhooks**

- In GitHub → Settings → Webhooks, add a webhook:
  - Payload URL: http://your-jenkins-server/github-webhook/
  - Content type: application/json
  - Select "Just the push event"


**Step 10: Validate Deployment**

- Run docker ps on the EC2 instance to ensure the container is running.
- Test the application by visiting the EC2 public IP.




**Final Architecture**

1. Developer pushes code to GitHub.
2. Jenkins pulls code, builds Docker image, and pushes it to DockerHub.
3. EC2 pulls the Docker image and deploys the container.
4. AWS CloudWatch monitors logs and application health.
