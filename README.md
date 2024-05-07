# CI-CD-pipeline using EKS
![image](https://github.com/Ritish134/EKS-CI-CD-pipeline/assets/121374890/dc425fc0-2013-4366-a4a9-eb7ad7de0cd2)


Firstly create ec2 ubuntu machine for jenkins master (15Gb HDD) : 
- sudo apt update
- sudo apt updgrade
- sudo nano /etc/hostname (to change the hostname)
- sudo init 6
- Add inbound rule port 8080
- sudo apt install openjdk-17-jre
#### From official docs :
```
  sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```
- sudo systemctl enable jenkins
- sudo systemctl start jenkins
- sudo systemctl status jenkins
  
### Create new ec2 machine callled jenkins agent ubuntu (15GB HDD)
- sudo apt update
- sudo apt updgrade
- sudo nano /etc/hostname (to change the hostname)
- sudo init 6
- sudo apt install openjdk-17-jre
- sudo apt-get install docker.io
- sudo usermod -aG docker $USER ( to give access to current user of group docker)
- sudo init 6
- sudo nano /etc/ssh/sshd_config
- uncomment publickeauthenticatio and authorizekeyfile
- Do same on the jenkins master as well
- Also (sudo service sshd reload) on both
  ### Go to jenkins master
  - ssh-keygen
  - cd .ssh
  - copy pub key file content
  - and paste it on jenkins agent inside .ssh/authorizedkeys
  - copy public ip of jenkins master and open it with port 8080 ( install suggested plugins)
  - go to manage jenkins > nodes > built-in nodes > configure > number of executor =0 , save 
  - go to managee jenkins > nodes > new node give name jenkins-agent and tick permanent > number of executors=2 , /home/ubuntu, label=jenkins-agent, launch agent via ssh , in host paste private IP of agent
  - add credential > ssh username with private key , id=jenkins-agent , username=ubuntu , enter directly paste the private key from master node , credential take ubuntu, host key verfication - non verifying
  - Connectivity bwtween master and agent is success
 
    ## Integrate Maven to jenkins
    - Go to available plugins - maven,pipeline maven, eclipse temurin
    - Manage jenkins > tools > Add maven give maven3, install automatically > save
    - Manage jenkins > tools > Add java give java17, install automatically,install from adaptium.net jdk-17.05+8 > save
    - manage jenkins > credentials > add your github credentials with id of github

    ## Create jenkinsfile
    ```
    pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
	    APP_NAME = "simple-app-pipeline"
            RELEASE = "1.0.0"
            DOCKER_USER = "ritish134"
            DOCKER_PASS = 'dockerhub'
            IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
            IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
	    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }
    stages{
        stage("Cleanup Workspace"){
                steps {
                cleanWs()
                }
        }

        stage("Checkout from SCM"){
                steps {
                    git branch: 'main', credentialsId: 'github', url: 'https://github.com/ritish134/simple-app'
                }
        }

        stage("Build Application"){
            steps {
                sh "mvn clean package"
            }

       }

       stage("Test Application"){
           steps {
                 sh "mvn test"
           }
       }
  }
}

- Go to jenkins dashboard > new item > simple-app-ci choose pipeline
- tick discard old build , builds to keep=2 , pipleine script from scm , scm = git choose github credentials , build now

## Install and configure Sonarqube
- create a new ec2 ubuntu machine of t3.medium 15GB hdd
- sudo apt update
- sudo apt upgrade
### Add PostgresSQL repository
     sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
     wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
### Install PostgreSQL
     sudo apt update
     sudo apt-get -y install postgresql postgresql-contrib
     sudo systemctl enable postgresql
### Create Database for Sonarqube
     sudo passwd postgres
     su - postgres
     createuser sonar
     psql 
     ALTER USER sonar WITH ENCRYPTED password 'sonar';
     CREATE DATABASE sonarqube OWNER sonar;
     grant all privileges on DATABASE sonarqube to sonar;
     \q
     exit
### Add Adoptium repository
     sudo bash
     wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
     echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
 ### Install Java 17
     apt update
     apt install temurin-17-jdk
     update-alternatives --config java
     /usr/bin/java --version
     exit
## Linux Kernel Tuning
   # Increase Limits
     sudo vim /etc/security/limits.conf
    //Paste the below values at the bottom of the file
    sonarqube   -   nofile   65536
    sonarqube   -   nproc    4096

    # Increase Mapped Memory Regions
    sudo vim /etc/sysctl.conf
    //Paste the below values at the bottom of the file
    vm.max_map_count = 262144
- sudo init 6
- allow port 9000
#### Sonarqube Installation ####
## Download and Extract
    $ sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
    $ sudo apt install unzip
    $ sudo unzip sonarqube-9.9.0.65466.zip -d /opt
    $ sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube
## Create user and set permissions
     $ sudo groupadd sonar
     $ sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
     $ sudo chown sonar:sonar /opt/sonarqube -R
## Update Sonarqube properties with DB credentials
     $ sudo vim /opt/sonarqube/conf/sonar.properties
     //Find and replace the below values, you might need to add the sonar.jdbc.url
     sonar.jdbc.username=sonar
     sonar.jdbc.password=sonar
     sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
## Create service for Sonarqube
$ sudo vim /etc/systemd/system/sonar.service
//Paste the below into the file
     [Unit]
     Description=SonarQube service
     After=syslog.target network.target

     [Service]
     Type=forking

     ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
     ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

     User=sonar
     Group=sonar
     Restart=always

     LimitNOFILE=65536
     LimitNPROC=4096

     [Install]
     WantedBy=multi-user.target

## Start Sonarqube and Enable service
     $ sudo systemctl start sonar
     $ sudo systemctl enable sonar
     $ sudo systemctl status sonar

## Watch log files and monitor for startup
     $ sudo tail -f /opt/sonarqube/logs/sonar.log
Now copy public ip of sonarqube ec2 and open it at port 9000

## Integrate Sonarqube with jenkins
 - In sonarqube dashboard under security , generate new token
 - go to jenkins add credential , kind secret text
 - install sonarqube scanner,sonar quality gates,quality gates
 - manage jenkins > system , under add sonarqube
 - name - sonarqube server , url = private ip of sonarqube server with 9000 port
 - manage jenkins > tools > undersonarqube installations , name sonarqube-scanner> install automatically 5.01.3006

### Add one more stage in the jenkinsfile
```
stage("SonarQube Analysis"){
           steps {
	           script {
		        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
		        }
	           }	
           }
       }
```
Then build now in jenkins to check stage is added
- Create webhook in sonarqube gui
- paste https://<private-ip-of-jenkins-master>/sonarqube-webhook

  Add one more stage in jenkinsfile
  ```
    stage("Quality Gate"){
           steps {
               script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }	
            }

        }
## Build and push docker image using pipeline script
- Go to manage jenkins > plugins > docker , docker commons,docker pipeline,docker API,docker build step,cloud bees
- Add credential for docker hub with id dockerhub
- Add one more stage to jenkins
  ```
  	stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }

       }
  ```
  To scan the docker image use trivy
  in jenkins file
  ```
   	stage("Trivy Scan") {
           steps {
               script {
	            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ritish134/simple-app-pipeline:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
               }
           }
       }
  stage ('Cleanup Artifacts') {
           steps {
               script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
               }
          }
       }
  ```
  ## Setup bootstrap server for eksctl
  - create new ec2 eks-bootstrap server ubuntu
  - sudo apt update
  - sudo apt upgrade
  - sudo nano /etc/hostname
  - sudo init 6
  - install aws cli
  - 	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  - apt install unzip
  - unzip awscliv2.zip
  - sudo ./aws/install
  - Download kubectl
  - curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
  - chmod +x ./kubectl
  - mv kubectl /bin
    ## Installing  eksctl
	Refer---https://github.com/eksctl-io/eksctl/blob/main/README.md#installation
	 curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
	 cd /tmp
	 ll
	 sudo mv /tmp/eksctl /bin
	 eksctl version
    ## Setup Kubernetes using eksctl
	Refer--https://github.com/aws-samples/eks-workshop/issues/734
	 eksctl create cluster --name virtualtechbox-cluster \
	--region ap-south-1 \
	--node-type t2.small \
	--nodes 3 \
	
	 kubectl get nodes
  ## ArgoCD on EKS cluster
  - kubectl create namespace argocd
  -  Next, let's apply the yaml configuration files for ArgoCd
       kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  -  Now we can view the pods created in the ArgoCD namespace.
      kubectl get pods -n argocd
  -  To interact with the API Server we need to deploy the CLI:
     curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
     chmod +x /usr/local/bin/argocd
  - Expose argocd-server
    $ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

  - Wait about 2 minutes for the LoadBalancer creation
    $ kubectl get svc -n argocd

  - Get pasword and decode it.
    $ kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
    $ echo WXVpLUg2LWxoWjRkSHFmSA== | base64 --decode

  ## Add EKS Cluster to ArgoCD
- login to ArgoCD from CLI
    $ argocd login a2255bb2bb33f438d9addf8840d294c5-785887595.ap-south-1.elb.amazonaws.com --username admin

- 
     $ argocd cluster list

- Below command will show the EKS cluster
     $ kubectl config get-contexts

- Add above EKS cluster to ArgoCD with below command
     $ argocd cluster add i-08b9d0ff0409f48e7@virtualtechbox-cluster.ap-south-1.eksctl.io --name virtualtechbox-eks-cluster

- $ kubectl get svc
- Add github repo using argocd GUI

## Create CD job in jenkins to automate the process
- use project type pipeline >  check discard old builds , max build=2,this project is parameterized > string parameter , name=IMAGE_TAG, trigger builds remotely = gitops-token,pipeline script from scm and git
- Add more stage to jenkins file
  ```
  	stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
                }
            }
       }
    }

    post {
       failure {
             emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed", 
                      mimeType: 'text/html',to: "ashfaque.s510@gmail.com"
      }
      success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful", 
                     mimeType: 'text/html',to: "ashfaque.s510@gmail.com"
      }      
   }
}
- generate new api token on jenkins
- then add credentials in jenkins , kind secret text

============================================================= Cleanup =============================================================
$ kubectl get all
$ kubectl delete deployment.apps/virtualtechbox-regapp       //it will delete the deployment
$ kubectl delete service/virtualtechbox-service              //it will delete the service
$ eksctl delete cluster virtualtechbox --region ap-south-1     OR    eksctl delete cluster --region=ap-south-1 --name=virtualtechbox-cluster      //it will delete the EKS cluster

  
  

