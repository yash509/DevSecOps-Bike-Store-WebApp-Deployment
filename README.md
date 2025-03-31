Automated Deployment: Docker, Jenkins, and SonarQube for Static Websites. Bike-App-Store


<img width="960" alt="2025-02-19 (16)" src="https://github.com/user-attachments/assets/68ccf72c-ccf0-4346-8e4e-329c0f146a96" />


<img width="960" alt="2025-02-19 (11)" src="https://github.com/user-attachments/assets/15dcf622-5de8-4c80-9817-54e01b410a6e" />

**Bike-App-Store**
Clone the repo: https://github.com/etaoko333/Bike-App-Store.git

**Introduction**

In modern web development, automating deployment and ensuring your code quality is crucial for maintaining high standards and reducing errors. This blog post will walk you through the steps to automate the deployment of a static website using Docker and Jenkins, with added steps to integrate SonarQube for static code analysis and enforce a Quality Gate.

- By the end of this post, you’ll have an end-to-end CI/CD pipeline that:

****Builds and tests your website.
- Runs static analysis via SonarQube.
- Deploys the website to a Docker container.
- Why Use Docker, Jenkins, and SonarQube?
- Docker ensures your application runs consistently across different environments (local, staging, production).
- Jenkins automates the entire CI/CD pipeline — from code push to deployment.
- SonarQube helps you continuously analyze and measure code quality and technical debt, providing insights into potential issues like bugs, vulnerabilities, and code smells.
- Together, they create a powerful, automated, and scalable deployment pipeline.

**Benefits:**
- Portability: Docker ensures a consistent environment everywhere.
- Quality Assurance: SonarQube checks code quality before deployment.
- Efficiency: Jenkins automates testing, building, and deployment.

**Step 1: Prepare Your Static Website**
Before we start Dockerizing and automating deployment, ensure your static website is ready. 
-This website can be as simple as an HTML, CSS, and JavaScript project. We’ll use the Bike App Store as an example.

**Tools You’ll Need:**
- HTML/CSS/JS files: Your website files.
- GitHub: To store your project code and integrate with Jenkins.
- Ensure that the website is tested locally before moving to the next steps.

**Step 2: Dockerize Your Static Website**
The first step in the CI/CD pipeline is Dockerizing your static website so it can run in any environment.

**Tools:**
- Docker: To containerize your website.
- Docker Hub: To store and deploy Docker images.

**Steps:**
- Create a Dockerfile
- A Dockerfile describes the steps to create your Docker image. Here’s an example for a static website:
-A Dockerfile describes the steps to create your Docker image. Here’s an example for a static website:
 # Use Nginx image to serve static content

# Use Nginx image to serve static content 
```FROM nginx:alpine  # Copy website files to the nginx container 
COPY . /usr/share/nginx/html 
Expose port 80```

- Build the Docker Image Run the following command in your project directory to build the Docker image:
```docker build -t bike-app-store .```

Test the Docker Image To test if your website is working, run the container:
```docker run -d -p 8080:80 bike-app-store```
```Visit http://localhost:8080 to check the website.```

**Step 3: Set Up Jenkins for Continuous Deployment**
Jenkins is responsible for automating the entire pipeline — from pulling the code from GitHub to pushing the Docker image to production.

**Tools:**
- Jenkins: Automates the build, test, and deployment pipeline.
- GitHub: Stores the code.
- Docker Hub: Stores the Docker image.
- SonarQube: Provides static code analysis and enforces the quality gate.

**Steps:**
Install Jenkins If Jenkins is not installed, run the following commands:

``sudo apt update``
``sudo apt install openjdk-11-``
``wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee /etc/apt/trusted.gpg.d/jenkins.asc
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable/ / > /etc/apt/sources.list.d/jenkins.list'``

```sudo apt update```
```sudo apt install jenkins```
- Install Jenkins Plugins In Jenkins, install the following plugins to enable integration with GitHub, Docker, and SonarQube:
- Pipeline stage view
-Maven
- Eclipse Temurin
- Nodejs
- Extended Email
- Docker Plugin
- SonarQube Plugin
- Pipeline Plugin
- You can install these from Jenkins → Manage Jenkins → Manage Plugins.
- Create global creadentials
- Configure the tools
-configure the system

3. Create a New Jenkins Pipeline Create a new Pipeline project in Jenkins. Under the Pipeline Script section, you’ll define the pipeline that will automate the build, test, and deployment.

**Step 4: Write Your Jenkins Pipeline Script with SonarQube and Quality Gate**
Here’s how we can integrate SonarQube for static analysis and enforce a Quality Gate before proceeding to the deployment stage.

```Jenkinsfile:
groovy
pipeline {
    agent anytools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'  // SonarQube scanner tool
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()  // Clean the workspace
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/etaoko333/Bike-App-Store.git'  // Checkout code from GitHub
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {  // Use SonarQube environment
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Bike-App-Store \
                        -Dsonar.projectKey=Bike-App-Store
                    '''
                }
            }
        }
        stage('Quality Gate Check') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true  // Enforce Quality Gate
                    }
                }
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"  // Run Trivy for file system scan
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t bike-app-store ."
                        sh "docker tag bike-app-store sholly333/bike-app-store:latest"
                        sh "docker push sholly333/bike-app-store:latest"
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image sholly333/bike-app-store:latest > trivy.txt"  // Run Trivy for image scan
            }
        }
        stage('Deploy to Docker Container') {
            steps {
                sh '''
                docker stop bike-app-store || true
                docker rm bike-app-store || true
                docker run -d --name bike-app-store -p 8080:80 sholly333/bike-app-store:latest
                '''
            }
        }
    }
}```

**Explanation of the Stages:**
- Clean Workspace: Ensures Jenkins starts fresh every time.
- Checkout from Git: Pulls the latest code from GitHub.
- SonarQube Analysis: Runs SonarQube analysis on your code.
- Quality Gate Check: Enforces the SonarQube Quality Gate to ensure the code meets defined quality standards. If the quality gate fails, the pipeline is aborted.
- Trivy FS Scan: Performs a file system scan to check for vulnerabilities.
- Docker Build & Push: Builds the Docker image and pushes it to Docker Hub.
- Trivy Image Scan: Scans the Docker image for vulnerabilities.
- Deploy to Docker Container: Deploys the website to a Docker container.

**Step 5: Set Up Continuous Deployment (CD)**
Once the Jenkins pipeline is set up, Jenkins will automatically trigger a build every time there’s a new commit in your GitHub repository. The pipeline will:

- Run SonarQube analysis.
- Enforce the Quality Gate.
- Build and deploy the Docker image.
**Conclusion**
By integrating Docker, Jenkins, and SonarQube, you’ve built an automated CI/CD pipeline that ensures code quality and seamless deployment. This setup saves time, reduces human error, and ensures that only code that passes quality checks gets deployed.

**Key Takeaways:**
- Docker provides a consistent environment for your static website.
- Jenkins automates the CI/CD pipeline from build to deployment.
-SonarQube ensures code quality by analyzing and enforcing the Quality Gate.
- You’ve automated the entire process, from code push to production deployment.
- With this setup, you can confidently scale and deploy your static websites with high standards. 🚀
