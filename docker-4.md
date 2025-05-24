# Docker Volume-Based Multi-Branch Deployment via Jenkins Slave Node

**Assignment:** Docker | 4
**Author:** Vinay Hinukale

---

## Objective

Deploy three versions of an `index.html` webpage from three GitHub branches (`2025Q1`, `2025Q2`, `2025Q3`) into three Docker `httpd` containers using volumes. Each container will serve on a unique host port (**80**, **90**, **8001**) and will be deployed using a Jenkins pipeline running on a Jenkins **slave node**.

---

## Jenkins Master Prerequisites

* Jenkins Master already installed and running
* SSH access and credentials to the slave EC2 instance

---

## Jenkins Slave Node Configuration

1. **Login to Jenkins Master UI**
   Navigate to: `Dashboard → Manage Jenkins → Nodes → New Node`

2. **Configure Node:**

   * **Name:** `slave`
   * **Type:** `Permanent Agent`
   * **# of Executors:** `2`
   * **Remote Root Directory:** `/home/ec2-user/Jenkins`

     ```bash
     # On Slave EC2 instance, Install git, java, docker
     mkdir -p /home/ec2-user/Jenkins
     sudo yum install java-17-amazon-corretto.x86_64 -y
     sudo yum install git -y
     sudo yum install docker -y
     sudo usermod -aG docker ec2-user
     ```
   * **Labels:** `slave`
   * **Usage:** Only build jobs with label expressions matching this node
   * **Launch Method:** Launch agents via SSH
   * **Host:** Private IP of slave EC2
   * **Credentials:**

      * **Type:** SSH Username with private key
      * **Scope:** Global
      * **Username:** `ec2-user` (or `jenkins` if configured with sudo)
      * **Private Key:** Paste `.pem` or `.ppk` directly
   * **Host Key Verification Strategy:** Manually trusted key verification

3. Apply and Save

4. Now go to node slave and accept the key and continue, now its connected to slave

---

## Jenkins Pipeline Job Configuration

1. **Create a New Pipeline Job**

   * **Name:** `docker_multibranch_slave_pipeline`
   * **Type:** `Pipeline`

2. **Paste the Following Pipeline Script:**

```groovy
pipeline {
    agent {
        label {
            label "slave"
        }
    }

    environment {
        REPO_URL = 'https://github.com/LegPro/docker-repo-1.git'
    }

    stages {
        stage("define-branches") {
            steps {
                script {
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
                sudo yum install docker -y
                sudo systemctl start docker
                docker stop $(docker ps -aq --filter name=Q) || true
                docker rm $(docker ps -aq --filter name=Q) || true
                '''
            }
        }

        stage("deploy-branches") {
            steps {
                script {
                    def basePath = "/home/ec2-user/Jenkins/workspace/${env.JOB_NAME}"

                    BRANCH_PORTS.eachWithIndex { branchEntry, idx ->
                        def branchName = branchEntry.key
                        def hostPort = branchEntry.value
                        def containerName = "Q${idx + 1}"
                        def localDir = "${branchName}"

                        echo "Cloning branch: ${branchName} into ${localDir}"
                        dir(localDir) {
                            git branch: branchName, url: env.REPO_URL
                            sh 'pwd && ls -l'
                        }

                        echo "Starting container ${containerName} on port ${hostPort}"
                        sh """
                        docker run -itd --name ${containerName} -p ${hostPort}:80 \
                          -v ${basePath}/${localDir}:/usr/local/apache2/htdocs httpd
                        docker exec ${containerName} ls -l /usr/local/apache2/htdocs
                        docker exec ${containerName} chmod -R 777 /usr/local/apache2/htdocs
                        """
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

## Expected Outcome

* Each GitHub branch (`2025Q1`, `2025Q2`, `2025Q3`) is cloned on the slave node
* Corresponding `index.html` files are volume-mounted into Docker containers
* Containers are exposed at:

   * `http://<host>:80`
   * `http://<host>:90`
   * `http://<host>:8001`

---

## Conclusion

This documentation showcases how to integrate Docker volume usage with Jenkins slave nodes for automated multi-version web deployments across separate HTTPD containers. It enables isolated and reproducible CI/CD workflows for multi-branch repositories.
