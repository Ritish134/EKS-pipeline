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
	    APP_NAME = "register-app-pipeline"
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
```
