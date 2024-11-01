# DevSecOps CI/CD Pipeline with Deployment on Azure

### For detailed web application information, please refer to the `README` file in the `src` directory.

<br/>

## Setup Steps before running through Jenkins pipeline


1. **Install Jenkins, Docker, and Trivy**

  
2. **Create a SonarQube container using Docker and get a TMDB API Key**
    ```bash
    docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
    ```


3. **Install Prometheus and Grafana**
   - Set up using `nssm` locally or on an Ubuntu instance.
   - Install Node Exporter (or Windows Exporter if using Windows) and add it to the Prometheus configuration file (`prometheus.yml`) for detection.


4. **Integrate Prometheus with Jenkins**
   - Install the Prometheus Plugin in Jenkins and connect it to your Prometheus server.


5. **Email Integration with Jenkins**
   - Set up your Google Account and generate an App Password.
   - Install the email notification plugin.
   - Configure email notifications and add credentials.
   - Set up the extended email notification settings.


6. **Install Required Plugins in Jenkins**
   - Install plugins such as JDK, SonarQube Scanner, Node.js, and OWASP Dependency Check.

7. **Install Docker Related Plugins and Add DockerHub Credentials**
   - Eclipse Temurin Installer
   - Docker
   - Docker Commons
   - Docker Pipeline
   - Docker API
   - docker-build-step


9. **Build and Push Docker Image**


10. **Deploy the Docker Image**

<br>

## Further Steps for deployment
1. **Configure Azure and Deploy Resources with Terraform**
   - Install terraform
   - Set up Azure (or your chosen cloud provider) to use Terraform for deploying resources.
   - After logging into Azure with the Azure CLI (`az login`), run the following commands:

     ```bash
     terraform init
     
     terraform plan

     terraform apply
     ```

   - (Optional) Deploy an Azure Container Registry (ACR) to store your Docker image.
     - Only if ACR deployment fails, manually push the image to ACR.


3. **Deploy the App Image Using Kubernetes**
   - Use Kubernetes to deploy the Docker image from ACR to Azure Kubernetes Service (AKS) using a deployment YAML file.
   - Open PowerShell in Azure and execute the following commands:

     ```bash
     az "dns_prefix" get-credentials --resource-group "resource_group_name" --name "aks_name"

     kubectl apply -f deployment.yml

     kubectl get service "service-name" --watch
     ```

   - This will provide you with the external IP for your application, which you can access through a browser.

<img src="https://github.com/user-attachments/assets/ccef8996-9319-47e4-9a2f-a21f548159aa" alt="Front page" width="1000">

<br>

## Jenkinsfile

Here’s the complete pipeline for Jenkins:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/ChrisDc777/devsecops-prufen.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    bat ''' %SCANNER_HOME%\\bin\\sonar-scanner -D"sonar.projectName=Netflix" \
                    -D"sonar.projectKey=Netflix" '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                bat "npm install"
            }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                bat "trivy fs . > trivyfs.txt"
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        bat "docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix ."
                        bat "docker tag netflix your-docker-name/netflix:latest"
                        bat "docker push your-docker-name/netflix:latest"
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                bat "trivy image your-docker-name/netflix:latest > trivyimage.txt"
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'your-emailid-configured',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```

<br/>

<img src="https://github.com/user-attachments/assets/8269fd4f-7b63-4d0c-8cd3-e4b244e2eb9c" alt="Front page" width="1000">

