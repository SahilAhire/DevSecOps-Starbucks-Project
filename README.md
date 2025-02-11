![Starbucks Clone Deployment](https://github.com/SahilAhire/DevSecOps-Starbucks-Project/blob/main/Finale%20flow%20diagram.png)
 
# **Install Jenkins on Ubuntu:**

```
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```


# **Install Docker on Ubuntu:**
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker
sudo systemctl status docker
```

# **Install Trivy on Ubuntu:**

Reference Doc: https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/
```
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

# Jenkins Complete pipeline
```
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_REGISTRY = 'atios2309'
        IMAGE_NAME = 'starbucks'
    }
    stages {
        stage ("Clean Workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/SahilAhire/DevSecOps-Starbucks-Project.git'
            }
        }
        stage ("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=starbucks \
                        -Dsonar.projectKey=starbucks'''
                }
            }
        }
        stage ("Quality Gate Check") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'Sonar-token'
                }
            }
        }
        stage ("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage ("Trivy Security Scan") {
            steps {
                sh "trivy fs . > trivy.txt"   // Run Trivy scan and output to file
                sh "cat trivy.txt"           // Display the results in Jenkins console
            }
        }
        stage ("Build Docker Image") {
            steps {
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }
        stage ("Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag ${IMAGE_NAME}:latest ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        stage ("Deploy to Docker Swarm") {
            steps {
                sh '''
                docker swarm init --advertise-addr 10.0.2.15
                docker service create --name starbucks --replicas 5 \
                    --publish 3000:3000 \
                    --restart-condition any \
                    atios2309/starbucks:latest
                '''
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}' - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <html>
                    <body>
                        <h2 style="color: #4CAF50;">Jenkins Build Report</h2>
                        <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><strong>URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    </body>
                    </html>
                """,
                to: 'sahil.ahire2000@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt'
        }
    }
}


```

## Jenkins Pipeline Work Flow
![Jenkin](https://github.com/hrishi-d-d/Starbucks/blob/main/Screenshot%20(445).png)


## SonarQube Analysis Report
![SonarQube](https://github.com/hrishi-d-d/Starbucks/blob/main/Screenshot%20(444).png)


## Starbucks Clone 
![Jenkin](https://github.com/hrishi-d-d/Starbucks/blob/main/image.png)

