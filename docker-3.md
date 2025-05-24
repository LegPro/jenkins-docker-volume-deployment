# Docker Volume-Based Multi-Branch Deployment via Jenkins Pipeline

**Assignment:** Docker | 3
**Author:** Vinay Hinukale

---

## Objective

Deploy three versions of an `index.html` webpage, each from a separate GitHub branch (`2025Q1`, `2025Q2`, `2025Q3`),
into three `httpd` containers using Docker volumes. These containers are launched and managed through a Jenkins
multi-branch pipeline. Each container is exposed on a different host port: **80**, **90**, and **8001**.

---

## Jenkins Master Configuration

1. **Launch EC2 Instance** (Amazon Linux 2)
2. **Install Jenkins**

   ```bash
   sudo su
   sudo wget -O /etc/yum.repos.d/jenkins.repo \
       https://pkg.jenkins.io/redhat-stable/jenkins.repo

   sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
   yum install java-17-amazon-corretto.x86_64 -y
   yum install git -y
   yum install jenkins -y
   ```
3. **Configure Jenkins User**

   ```bash
   visudo
   # Add this line:
   jenkins ALL=(ALL)       NOPASSWD:ALL
   ```
4. **Start Jenkins and Access UI**

   ```bash
   service jenkins start
   cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
5. **Allow Jenkins User Docker Access**

   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```
6. **Finish Jenkins Setup**

    * Go to: `http://<EC2_PUBLIC_IP>:8080`
    * Complete the Jenkins UI setup.

---

## Pipeline Setup

1. **Create a New Pipeline Job**

    * **Name:** `multi_branch_docker_deployment`
    * **Type:** Pipeline

2. **Configure Pipeline Script**

    * Paste your Jenkins pipeline Groovy script.

   ```
   pipeline {
    agent {
        label {
            label "built-in"
        }
    }

    environment {
        REPO_URL = 'https://github.com/LegPro/docker-repo-1.git'
    }

    stages {
        stage("define-branches") {
            steps {
                script {
                    // Define branches and corresponding ports
                    BRANCH_PORTS = [
                        '2025Q1': 80,
                        '2025Q2': 90,
                        '2025Q3': 8001
                    ]
                }
            }
        }

        stage("docker-install") {
            steps {
                sh '''
                echo "Installing Docker..."
                sudo yum install docker -y
                sudo systemctl start docker
                docker stop Q1 Q2 Q3 || true
                docker rm Q1 Q2 Q3 || true
                '''
            }
        }

        stage("deploy-branches") {
            steps {
                script {
                    def basePath = "${env.WORKSPACE}"
                    def counter = 1

                    BRANCH_PORTS.each { branchName, hostPort ->
                        def containerName = "Q${counter}"
                        def localDir = "${branchName}"

                        echo "Cloning branch: ${branchName} into ${localDir}"
                        dir(localDir) {
                            git branch: branchName, url: env.REPO_URL
                        }

                        echo "Starting container ${containerName} on port ${hostPort}"
                        sh """
                        docker run -itd --name ${containerName} -p ${hostPort}:80 \
                          -v ${basePath}/${localDir}:/usr/local/apache2/htdocs httpd
                        docker exec ${containerName} chmod -R 777 /usr/local/apache2/htdocs
                        """

                        counter++
                    }
                }
            }
        }
    }
   }
   ```
    * ✅ Tick: **Use Groovy Sandbox**
    * 💾 Click: **Apply and Save**

---

## Expected Result

* Jenkins clones 3 different branches:

    * `2025Q1`, `2025Q2`, `2025Q3`
* Each branch's `index.html` is deployed using Docker volume mounting.
* Containers serve on:

    * `http://<host>:80` (for branch `2025Q1`)
    * `http://<host>:90` (for branch `2025Q2`)
    * `http://<host>:8001` (for branch `2025Q3`)

---

## Conclusion

This setup demonstrates **Docker volume** usage combined with **Jenkins pipelines** to automate deployment from multiple
Git branches to separate container instances. It is an efficient method to preview or test multiple versions of a
website or application in isolation.
