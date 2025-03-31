pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "yash5090/bmical"
        TAG = "${params.DOCKER_TAG}" 
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_ACCOUNT_ID="992382397067"
        AWS_DEFAULT_REGION="ap-south-1"
        IMAGE_REPO_NAME="webapp-image"
        IMAGE_TAG="v1"
        REPOSITORY_URI = "992382397067.dkr.ecr.ap-south-1.amazonaws.com/webapp-image"
		VAULT_URL = 'http://13.233.207.160:8200/' // Vault server URL needs to be changed everytime manually
    }
    
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'jenkins --version'
                sh 'aws --version'
                sh 'kubectl version --client'
                sh 'terraform --version'
                sh 'trivy --version'
                sh 'docker --version'
                sh 'ansible --version'
                sh 'snyk --version'
				sh 'vault --version'
            }
        }
		
		stage("Debugging HashiCorp Vault Credentials") {
            steps {
                script {
                    echo "Verifying Vault Credentials Configuration..."
                    withCredentials([
                        string(credentialsId: 'VAULT_URL', variable: 'VAULT_URL'),
                        string(credentialsId: 'vault-role-id', variable: 'VAULT_ROLE_ID'),
                        string(credentialsId: 'vault-secret-id', variable: 'VAULT_SECRET_ID')
                    ]) {
                        echo "Vault Role ID is available"
                        echo "Vault Secret ID is available"
                    }
                }
            }
        }

        stage("Testing Hashicorp Vault Connectivity and Login") {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'VAULT_URL', variable: 'VAULT_URL'),
                        string(credentialsId: 'vault-role-id', variable: 'VAULT_ROLE_ID'),
                        string(credentialsId: 'vault-secret-id', variable: 'VAULT_SECRET_ID')
                    ]) {
                        echo "Testing Vault Connectivity..."
                        sh '''
                        # Set Vault address
                        export VAULT_ADDR="${VAULT_URL}"

                        # Log into Vault using AppRole
                        echo "Logging into Vault using AppRole..."
                        VAULT_TOKEN=$(vault write -field=token auth/approle/login role_id=${VAULT_ROLE_ID} secret_id=${VAULT_SECRET_ID})
                        echo "Vault Login Successful"

                        # Verify connectivity
                        vault status
                        '''
                    }
                }
            }
        }
        
        stage("Fetching Credentials from HashiCorp Vault") {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'VAULT_URL', variable: 'VAULT_URL'),
                        string(credentialsId: 'vault-role-id', variable: 'VAULT_ROLE_ID'),
                        string(credentialsId: 'vault-secret-id', variable: 'VAULT_SECRET_ID')
                        ]) 
                        {
                            echo "Fetching GitHub and AWS credentials from Vault..."
                            // Fetch secrets with error handling
                            sh '''
                            # Set Vault server URL
                            export VAULT_ADDR="${VAULT_URL}"
                            
                            # Log into Vault using AppRole
                            echo "Logging into Vault..."
                            VAULT_TOKEN=$(vault write -field=token auth/approle/login role_id=${VAULT_ROLE_ID} secret_id=${VAULT_SECRET_ID} || { echo "Vault login failed"; exit 1; })
                            export VAULT_TOKEN=$VAULT_TOKEN
                            
                            # Fetch GitHub token
                            # echo "Fetching GitHub Token..."
                            # GIT_TOKEN=$(vault kv get -field=pat secret/github || { echo "Failed to fetch GitHub token"; exit 1; })
                            
                            # Fetch AWS credentials
                            echo "Fetching AWS Credentials..."
                            AWS_ACCESS_KEY_ID=$(vault kv get -field=aws_access_key_id aws/terraform-project || { echo "Failed to fetch AWS Access Key ID"; exit 1; })
                            AWS_SECRET_ACCESS_KEY=$(vault kv get -field=aws_secret_access_key aws/terraform-project || { echo "Failed to fetch AWS Secret Access Key"; exit 1; })
                            
                            # Export credentials to environment variables
                            echo "Exporting credentials to environment..."
                            echo "export GIT_TOKEN=${GIT_TOKEN}" >> vault_env.sh
                            echo "export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}" >> vault_env.sh
                            echo "export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}" >> vault_env.sh
                            '''
                            // Load credentials into environment
                            sh '''
                            echo "Loading credentials into environment..."
                            . ${WORKSPACE}/vault_env.sh
                            echo "Credentials loaded successfully."
                            '''
                        }
                }
            }
        }
        
        stage('Docker Authentication with ECR') {
            steps {
                script {
                sh """aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"""
                }
                 
            }
        }
        
        stage('Checkout from Git') {                        
            steps {                                       
                git branch: 'main', url: 'https://github.com/yash509/DevSecOps-Bmicalculator-WebApp-Deployment.git'
            }
        }
        
        stage('Deployments') {
            parallel {
                stage('Test deploy to staging') {
                    steps {
                        echo 'staging deployment done'
                    }
                }
                stage('Test deploy to production') {
                    steps {
                        echo 'production deployment done'
                    }
                }
            }
        }
        
        stage('Test Build') {
            steps {
                echo 'Building....'
            }
            post {
                always {
                    jiraSendBuildInfo site: 'clouddevopshunter.atlassian.net'
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to Staging from main....'
            }
            post {
                always {
                    jiraSendDeploymentInfo environmentId: 'us-stg-1', environmentName: 'us-stg-1', environmentType: 'staging', issueKeys: ['JIRA-1234']
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to Production from main....'
            }
            post {
                always {
                    jiraSendDeploymentInfo environmentId: 'us-prod-1', environmentName: 'us-prod-1', environmentType: 'production', issueKeys: ['JIRA-1234']
                }
            }
        }
        
        stage("SonarQube Code Analysis ") {                         
            steps {
                //dir('Band Website') {
                    withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=web-application \
                    -Dsonar.projectKey=web-application'''
                    //}
                }
            }
        }
        
        stage("Quality Gate") {
            steps {
                //dir('Band Website') {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                    //}
                }
            }
        }
        
        stage('OWASP File System SCAN') {
            steps {
                //dir('Band Website') {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                //}
            }
        }
        
        stage('TRIVY File System SCAN') {
            steps {
                //dir('Band Website') {
                    sh "trivy fs . > trivyfs.txt"
                //}
            }
        }
        
        stage('Docker Scout Image Overview') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview' //docker scout cves local://breaking-bad:latest
                   }
                }   
            }
        }
        
        stage('Docker Scout CVES File System Scan') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout cves'
                   }
                }   
            }
        }

        stage('Snyk Loaded Vulnerability Test') {
        steps {
                withCredentials([string(credentialsId: 'snyk', variable: 'snyk')]) {
                   sh 'snyk auth $snyk'
                   sh 'snyk test --all-projects --report || true'
                   sh 'snyk code test --json-file-output=vuln1.json > snykloadedvulnerabilityreport.txt || true '
                }   
            }
        }
        
        stage('ECR Deployment Artifact Creation') {
            steps{
                script {
                    dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }
        
        stage("Docker Image Building"){
            steps{
                script{
                    //dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                            sh "docker build -t ${IMAGE_NAME}:${TAG} ." 
                            
                        //}
                    }
                }
            }
        }
        
        stage("Docker Image Tagging"){
            steps{
                script{
                    //dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                            sh "docker tag ${IMAGE_NAME}:${TAG} ${IMAGE_NAME}:${TAG}" 
                        //}
                    }
                }
            }
        }
        
        stage('Docker Image Scanning') { 
            steps { 
                sh "trivy image --format table -o trivy-image-report.html ${IMAGE_NAME}:${TAG}" 
            } 
        } 
        
        stage("Image Push to DockerHub") {
            steps{
                script{
                    //dir('Band Website') {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh "docker push ${IMAGE_NAME}:${TAG}"
                        //}
                    }
                }
            }
        }
        
        stage('Publishing Docker Image to Container Registry') {
            steps{  
                script {
                    sh """docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:$IMAGE_TAG"""
                    sh """docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"""
                }
            }
        }
        
        stage('Docker Scout Image Scanning') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker-scout quickview ${IMAGE_NAME}:${TAG}"
                       sh "docker-scout cves ${IMAGE_NAME}:${TAG}"
                       sh "docker-scout recommendations ${IMAGE_NAME}:${TAG}"
                       sh "docker-scout attestation ${IMAGE_NAME}:${TAG}"
                   }
                }   
            }
        }  
        
		stage('Snyk Docker Image/Container Vulnerability Scannning') {
		steps {
                withCredentials([string(credentialsId: 'snyk', variable: 'snyk')]) {
                   sh 'snyk auth $snyk'
                   sh 'snyk container test ${IMAGE_NAME}:${TAG} > snyk-containertest-vulnerabilityreport.txt --report || true '
                   sh 'snyk container monitor ${IMAGE_NAME}:${TAG} > snyk-containermonitor-vulnerabilityreport.txt || true'
                }
            }
        } 
		
        stage("TRIVY"){
            steps{
                //dir('Band Website') {
                    sh "trivy image ${IMAGE_NAME}:${TAG} > trivyimage.txt"   
                //}
            }
        }
		
		stage("Anchore Grype Docker Image Vulnerability Scan"){
            steps{
                sh "docker run --rm anchore/grype:v0.82.0 ${IMAGE_NAME}:${TAG}"
                sh "docker run --rm anchore/grype:v0.82.0 -o json ${IMAGE_NAME}:${TAG} > grypescanreport.json"
            }
        }
        
        stage('Docker Scout Artifacts Analysis') {
            steps {
                script{
                  withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                      sh "docker-scout sbom --format list ${IMAGE_NAME}:${TAG}" // docker-scout sbom ${IMAGE_NAME}:${TAG} :latest
                    }
                }   
            }
        }
        
        stage ('Manual Approval'){
          steps {
           script {
             timeout(time: 10, unit: 'MINUTES') {
              approvalMailContent = """
              Project: ${env.JOB_NAME}
              Build Number: ${env.BUILD_NUMBER}
              Go to build URL and approve the deployment request.
              URL de build: ${env.BUILD_URL}
              """
             mail(
             to: 'clouddevopshunter@gmail.com',
             subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", 
             body: approvalMailContent,
             mimeType: 'text/plain'
             )
            input(
            id: "DeployGate",
            message: "Deploy ${params.project_name}?",
            submitter: "approver",
            parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')])  
            }
           }
          }
        }

        stage ("Remove Docker Container") {
            steps{
                sh "docker stop bmical | true"
                sh "docker rm bmical | true"
             }
        }
        
        stage('Deploy to Docker Container'){
            steps{
                //dir('BMI Calculator (JS)') {
                    sh "docker run -d --name bmical -p 5000:80 ${IMAGE_NAME}:${TAG}" 
                //}
            }
        }

        stage ("Verify the Docker Deployments") {
            steps{
                sh "docker images -a"
                sh "docker ps -a"
             }
        }
        
        stage ('Snyk Kubernetes-Config Files Vulnerablity Scan') {
            steps {
                sh 'snyk iac test app-deployment-blue.yaml --report > BlueEnv-filevulnerabilityreport.txt || true'
                sh 'snyk iac test app-deployment-green.yaml --report > GreenEnv-filevulnerabilityreport.txt || true'
            }
        }

        stage("Sanity Check for Shifting to Production") {
            steps {
                input "Should we ship to prod?"
            }
        }
        
        stage('Deploy to Kubernetes'){
            steps{
                script{
                    //dir('K8S') {
                        def deploymentFile = ""
                        if (params.DEPLOY_ENV == 'blue') {
                            deploymentFile = 'app-deployment-blue.yaml'
                        } else {
                            deploymentFile = 'app-deployment-green.yaml'
                        }
                        
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh "kubectl apply -f ${deploymentFile}"
                                sh 'kubectl apply -f service.yaml'
                                sh 'kubectl apply -f job.yaml'
                                sh 'kubectl apply -f pv.yaml'
                                sh 'kubectl apply -f pvc.yaml'
                                sh 'kubectl apply -f nwp.yaml'
                                sh 'kubectl apply -f ingress.yaml'
                        //}
                    }
                }
            }
        }

        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        sh '''
                            kubectl patch service bmical-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bmical\\", \\"version\\": \\"''' + newEnv + '''\\"}}}"
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify the Kubernetes Deployments') { 
            steps { 
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') { 
                    sh "kubectl get all "
                    sh "kubectl get pods " 
                    sh "kubectl get svc "
                    sh "kubectl get ns"
                } 
            } 
        }
              
        stage('Deployment Done') {
            steps {
                echo 'Deployed Succcessfully...'
            }
        }
		
		stage('Dynamic Application Security Testing - OWASP ZAP'){
            steps {
                sh "docker run -v \$(pwd):/zap/wrk/:rw --network=\"host\" zaproxy/zap-stable zap-baseline.py -t http://13.232.80.132:8086 || true"
                sh "docker run -v \$(pwd):/zap/wrk/:rw --network=\"host\" zaproxy/zap-stable zap-baseline.py -t http://13.232.80.132:8080 || true"
                sh "docker run -v \$(pwd):/zap/wrk/:rw --network=\"host\" zaproxy/zap-stable zap-baseline.py -t http://13.232.80.132:9000 || true"
                sh "docker run -v \$(pwd):/zap/wrk/:rw --network=\"host\" zaproxy/zap-stable zap-baseline.py -t http://13.232.80.132:8200 || true"
                sh "docker run -v \$(pwd):/zap/wrk/:rw --network=\"host\" zaproxy/zap-stable zap-baseline.py -t http://13.232.80.132:5000 > penetration-testing-report.txt|| true"
            }
        }
    }
    
    post { 
        always {
            script { 
                def jobName = env.JOB_NAME 
                def buildNumber = env.BUILD_NUMBER 
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN' 
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red' 
                def body = """ 
                <html> 
                <body> 
                <div style="border: 4px solid ${bannerColor}; padding: 10px;"> 
                <h2>${jobName} - Build ${buildNumber}</h2> 
                <div style="background-color: ${bannerColor}; padding: 10px;"> 
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3> 
                </div> 
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p> 
                </div> 
                </body> 
                </html> 
            """ 
 
            emailext (
                attachLog: true,
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}", 
                body: body, 
                to: 'clouddevopshunter@gmail.com', 
                from: 'jenkins@example.com', 
                replyTo: 'jenkins@example.com', 
                mimeType: 'text/html', 
                attachmentsPattern: 'trivy-image-report.html, trivyfs.txt, trivyimage.txt, snykloadedvulnerabilityreport.txt,GreenEnv-filevulnerabilityreport.txt,BlueEnv-filevulnerabilityreport.txt, snyk-containertest-vulnerabilityreport.txt, snyk-containermonitor-vulnerabilityreport.txt, penetration-testing-report.txt')
				
				def buildDuration = currentBuild.durationString.replace(' and counting', '')
                def changes = currentBuild.changeSets.collect { changeSet ->
                    changeSet.items.collect { entry ->
                        "*${entry.author}:* ${entry.msg} (${entry.commitId})"
                    }.join('\n')
                }.join('\n')

                def buildUser = env.BUILD_USER ?: 'N/A'
                def buildUserEmail = env.BUILD_USER_EMAIL ?: 'N/A'

                slackSend(
                    channel: '#cloud_devsecops_engineer',
                    color: currentBuild.currentResult == 'SUCCESS' ? 'good' : 'danger',
                    message: """
                    *${currentBuild.currentResult}:* Job ${env.JOB_NAME} (${env.BUILD_NUMBER})
                    *Duration:* ${buildDuration}
                    *Started by:* ${currentBuild.getBuildCauses()[0].shortDescription}
                    *User Email:* ${env.BUILD_USER_EMAIL}
                    *Changes:*
                    ${changes}
                    *Workspace:* ${env.WORKSPACE}
                    *Node:* ${env.NODE_NAME}
                    *Jenkins URL:* ${env.JENKINS_URL}
                    *Executor Number:* ${env.EXECUTOR_NUMBER}
                    *Job URL:* ${env.JOB_URL}
                    *Build URL:* ${env.BUILD_URL}
                    *Build Timestamp:* ${new Date(currentBuild.startTimeInMillis).format("yyyy-MM-dd HH:mm:ss")}
                    *Build Parameters:* ${params}
                    *Build ID:* ${env.BUILD_ID}
                    *Build Tag:* ${env.BUILD_TAG}
                    *Build Display Name:* ${currentBuild.displayName}
                    *Build Executor:* ${env.EXECUTOR_NUMBER}
                    *Slave/Node Name:* ${env.NODE_NAME}
                    *Node Labels:* ${env.NODE_LABELS}
                    """
                )
            } 
        } 
    }
}

stage('Result') {
        timeout(time: 10, unit: 'MINUTES') {
            mail to: 'clouddevopshunter@gmail.com',
            subject: "${currentBuild.result} CI: ${env.JOB_NAME}",
            body: "Project: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nGo to ${env.BUILD_URL} and approve deployment"
            input message: "Deploy ${params.project_name}?", 
            id: "DeployGate", 
            submitter: "approver", 
            parameters: [choice(name: 'action', choices: ['Success'], description: 'Approve deployment')]
        }
    } 	