# Automating Docker container deployment with Jenkins :whale: 

## TL;DR
**WHAT**: Use Docker Plugins to connect Jenkins to Docker and automate entire containerisation process 

**HOW**: Install Jenkins, create a Docker host with AWS EC2 Instances, set up Docker Remote API , create Docker agent cloud, slave build agent images, installing and configuring Docker plugins

> [What is Docker Slave?](dockerslave.md)
---

**Contents**
1. [Part I: Installing Java8 and Jenkins](#part-i-installing-java8-and-jenkins)
2. [Part II: Configuring Jenkins and managing Docker plugins](#part-iii-configuring-jenkins)
3. [Part III: Setting a CI job](#setting-up-ci-build-on-jenkins)




---
## Part I Installing Java8 and Jenkins

> Full installation instructions [here](https://www.macminivault.com/installing-jenkins-on-macos/)

1. Install Jenkins on MacOS using Homebrew package manager :beers: 
> If homebrew already installed (run in Termainl `brew -v` to check), skip to the next step 
```bash
/usr/bin/ruby -e /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

2. Check if Homebrew requires any recommendations:
```bash
brew doctor
```

3. Before installing Jenkins, we need to install Java8
```bash
brew cask install java8
```
**Troubleshooting error**:
```bash
Cask 'java8' is unavailable: No Cask with this name exists.
java --version
java 14.0.1 2020-04-14
Java(TM) SE Runtime Environment (build 14.0.1+7)
Java HotSpot(TM) 64-Bit Server VM (build 14.0.1+7, mixed mode, sharing)
``` 

Jenkins requires Java8. Click [here](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) to install.

Check all installed java version
```bash
/usr/libexec/java_home -V
```
**which returns**
```bash
Matching Java Virtual Machines (2):
    14.0.1, x86_64:	"Java SE 14.0.1"	/Library/Java/JavaVirtualMachines/jdk-14.0.1.jdk/Contents/Home
    1.8.0_251, x86_64:	"Java SE 8"	/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```

Pick `1.8.0_251, x86_64` to set as the default then:
```bash
export JAVA_HOME=`/usr/libexec/java_home -v 1.8.0_251, x86_64`
```

When you run java -version you will see:
```bash
java version "1.8.0_251"
```

4. To make the Jenkins web interface accessible from anywhere, not just local machine, open up the config file:
```bash
sudo nano /usr/local/opt/jenkins-lts/homebrew.mxcl.jenkins-lts.plist
```

5. Find this line:
```bash
<string>--httpListenAddress=127.0.0.1</string>
```

6. Change it to:
```bash
<string>--httpListenAddress=0.0.0.0</string>
```
To exit out of nano, press `Ctrl+X`, hit `Y` to save Changes, hit `Enter`

7. Start or restart Jenkins
```bash
brew services start jenkins-lts
brew services restart jenkins-lts
```

8. Open the browser and type in the following:
```bash
http://localhost:8080/
```

9. To check whether jenkins is running:
```bash
brew services list
```

---
## Part II Configuring Jenkins
1. Navigate to Manage Jenkins > Manage Plugins > Available > Install **all** Docker plugins, `docker-build-step`, `Docker Compose Build Step`, `Docker build plugins` and `Github`

**List of plugins used:**
```bash
Blue Ocean
Credentials Plugin
Docker Plugin
Email Extension
Github Plugin
NodeJS Plugin
Oracle Java SE Development Kit Installer Plugin
Pipeline Plugin
Timestamper
Yet Another Docker Plugin
```

**Tip:** Install `Blue Ocean` plugin for better UI 
---

## Part III Setting up CI build on Jenkins 
1. Create a freestyle job on Jenkins and call it `Docker_Pipeline_Integration_Test` with the following configurations:
    - `General` -> `Discard Old Builds` -> `Max # of builds to keep` -> 3
    - `Github Project URL` -> Insert URL for Github Repository
    - `Source Code Management` -> `Git` ->  Insert `Repository URL` and `Credentials` (To learn how to add credentials click [here](https://sharadchhetri.com/how-to-setup-jenkins-credentials-for-git-repo-access/)) -> `Branches to build` -> `Branch Specifier` -> `*/dev*`
    - `Build Triggers` -> `GitHub hook trigger`
    -
    -
    -
    -

**Note**: Webhooks only work with public IP. You will need to forward your local port [http://localhost:8080/](http://localhost:8080/) to the Internet/public using an SSH server like [Serveo](https://medium.com/automationmaster/how-to-forward-my-local-port-to-public-using-serveo-4979f352a3bf), Ngrok or [SocketXP](https://www.socketxp.com/download). 

> To set up Github Webhooks, Jenkins, and Ngrok for Local Development click [here](https://medium.com/@developerwakeling/setting-up-github-webhooks-jenkins-and-ngrok-for-local-development-f4b2c1ab5b6)

2. On Github, navigate to your repository -> Go to `settings` -> `Webhooks` -> in Payload URL, enter Jenkins URL e.g:
```bash
http://naistangz-1234.socketxp.com/github-webhook/
```
-> Enable `SSL verification` -> `Update webhook` -> `Redeliver`

**Commands to forward local port to public IP with** `SocketXP`:
```bash
sudo su
sudo curl -O https://portal.socketxp.com/download/darwin/socketxp && chmod 777 socketxp && sudo mv socketxp /usr/local/bin
socketxp login "authentication_token_goes_here"
socketxp connect http://localhost:8080
Connected.
Public URL -> https://naistangz-z012h3op.socketxp.com
```

3. Go back to Jenkins and make sure `nodejs` plugin is installed
4. To activate `nodejs` plugin, go to `Manage Jenkins` > `System Configuration` > `Global Tool Configuration` > `NodeJS` > `Add NodeJS` > Give it a name e.g. `Node` > Save and Apply
5. Execute shell
```bash
cd app
npm install 
npm test
```
6. Click Apply and make changes on your IDE on a new branch and push to Github -> Jenkins will listen to incoming `POST` requests to the Payload URL used on Github and automatically merge changes from the new branch to the master branch if the tests pass. 

---

## Setting up CD build on Jenkins with Dockerfile 
- First go to [Dockerhub](https://hub.docker.com/r/naistangz), create an account if this has not been created yet, then set up a DockerHub repository and call it `<username>/<name_of_repository>` e.g `naistangz/docker_automation`
- Create a pipeline script on Jenkins
```json
pipeline {
  environment {
    registry = "naistangz/Docker_automation"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/naistangz/Docker_Jenkins_Pipeline'
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
         script {
            docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
  }
}
```
