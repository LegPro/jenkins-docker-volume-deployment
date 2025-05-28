# Amazon RDS | 1 | Jenkins-Based Deployment with RDS Integration

## Objective:

Deploy the LoginWebApp on Tomcat running inside a Docker container on Jenkins master, and configure its RDS connection using a Jenkins declarative pipeline.

---

## Pre-requisites

### 1. Jenkins Master Configuration with Required Components

```bash
yum install java-17-amazon-corretto.x86_64 -y
yum install git -y
yum install jenkins -y
```

**Install Docker & Maven 3.9.x**:

```bash
# Docker can be installed via Jenkins plugin or manually:
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker

# Maven setup
cd /opt
sudo curl -O https://downloads.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
sudo tar -xzvf apache-maven-3.9.6-bin.tar.gz
sudo mv apache-maven-3.9.6 /opt/maven

sudo tee /etc/profile.d/maven.sh > /dev/null <<EOF
export M2_HOME=/opt/maven
export PATH=\$M2_HOME/bin:\$PATH
EOF

sudo chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
mvn -v
```

**Configure Jenkins User**

```bash
visudo
# Add:
jenkins ALL=(ALL)       NOPASSWD:ALL
```

**Start Jenkins and Access UI**

```bash
service jenkins start
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Visit `http://<EC2_PUBLIC_IP>:8080` to finish setup.

**Allow Jenkins to Access Docker**

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 2. Launch RDS

* Launch a MySQL RDS instance
* Note its **Endpoint URL**, **Username**, and **Password**

---

## Step-by-Step: Use Jenkins Credentials Manager for DB Access

✅ **Add DB Credentials**:

* Go to: *Manage Jenkins → Credentials → Global → Add Credentials*
* **Kind**: Username with password
* **ID**: `rds-db-admin`
* **Username**: `admin`
* **Password**: `<your-RDS-password>`

---

## Pipeline Configuration

* Create new item → **Name**: `tomcat-loginapp-rds` → Select **Pipeline**

```groovy
pipeline {
    agent { label "built-in" }

    environment {
        IMAGE_NAME = "project-tomcat"
        CONTAINER_NAME = "project-container"
        GIT_REPO = "https://github.com/LegPro/project.git"
        MAVEN_HOME = '/opt/maven'
        PATH = "${MAVEN_HOME}/bin:${env.PATH}"
        DB_URL = 'jdbc:mysql://database-1.czqi002ku7kt.ap-south-1.rds.amazonaws.com:3306/test'
    }

    stages {
        stage("clone repository") {
            steps {
                git branch: 'master', url: "${env.GIT_REPO}"
            }
        }

        stage("Install Dependency") {
            steps {
                sh '''
                if ! command -v docker &> /dev/null
                then
                    echo "Docker not found, installing..."
                    sudo yum install docker -y
                    systemctl start docker
                    systemctl enable docker
                fi
                '''
            }
        }

        stage('Check Maven Version') {
            steps {
                sh 'mvn -v'
            }
        }

        stage('Clean Old Docker Image') {
            steps {
                sh '''
                docker rm -f ${CONTAINER_NAME} || true
                docker rmi -f ${IMAGE_NAME} || true
                '''
            }
        }

        stage('Build Project using Maven') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Run Docker Container') {
            environment {
                DB_CREDS = credentials('rds-db-admin')
            }
            steps {
                sh '''
                docker run -d -p 80:8080 \
                    -e DB_URL="$DB_URL" \
                    -e DB_USER="$DB_CREDS_USR" \
                    -e DB_PASS="$DB_CREDS_PSW" \
                    --name ${CONTAINER_NAME} ${IMAGE_NAME}
                '''
            }
        }
    }
 post {
        failure {
            echo "Build failed!"
        }
        success {
            echo "App deployed successfully at http://13.127.229.155/:80"
        }
    }
   
}
```

---

## MySQL Table Setup Script

Create `create_db_user_table.sh`

```bash
#!/bin/bash

RDS_HOST="url.amazonaws.com"
RDS_USER="user_name"
RDS_PASS="user_password"
DB_NAME="test"

mysql -h "$RDS_HOST" -u "$RDS_USER" -p"$RDS_PASS" -e "
CREATE DATABASE IF NOT EXISTS ${DB_NAME};
USE ${DB_NAME};
CREATE TABLE IF NOT EXISTS USER (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    password VARCHAR(100) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    regdate DATE DEFAULT (CURRENT_DATE())
);
"
```

```bash
chmod +x create_db_user_table.sh
./create_db_user_table.sh
```

---

## Dockerfile for Tomcat Deployment

```Dockerfile
FROM tomcat:9.0-jdk17

RUN rm -rf /usr/local/tomcat/webapps/ROOT

COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

---

## Verification

* Go to `http://<EC2_PUBLIC_IP>:80`
* Insert records from web UI
* Verify via CLI:

```bash
mysql -h <RDS-ENDPOINT> -u <username> -p
use test;
select * from USER;
```

---

## Conclusion

This pipeline automates the build, packaging, image creation, and deployment of LoginWebApp on Dockerized Tomcat. It securely connects the app to an RDS MySQL backend using Jenkins credentials, ensuring a clean CI/CD workflow.
