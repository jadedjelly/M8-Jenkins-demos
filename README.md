## Demo Project: 
### Create a CI Pipeline with Jenkinsfile (Freestyle, Pipeline, Multibranch Pipeline)

#### Technologies used:
- Jenkins, Docker, Linux, Git, Java, Maven

#### Project Description:
CI Pipeline for a Java Maven application to build and push to the repository
1. Create different Jenkins job types (Freestyle, Pipeline, Multibranch
pipeline) for the Java Maven project with Jenkinsfile to:
    - Connect to the application’s git repository
    - Build Jar
    - Build Docker Image
    - Push to private DockerHub repository

----------------------------------------------------------------------------------------

## Create pipeline job

- New item > give it a name "my-pipeline" > select pipeline > click ok
- Unlike freestyle jobs, pipelines uses scripts, from the dropdown menu, there's 2x options
    - pipeline script
    - pipeline script from scm (source code management)
    - You can view the samples from the right drop down menu:
```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

NOTE:
Below the script box, there's a check box called "groovy sandbox", it's tooltip describes it as:
"*If checked, run this Groovy script in a sandbox with limited abilities. If unchecked, and you are not a Jenkins administrator, you will need to wait for an administrator to approve the script. *"

- Best practice for IaC says that you should nt use inline pipeline scripts (scripts written directly into a pipeline) and should have it saved in your scm (understandable)

- we select "pipeline script from SCM"
    - input the git repo address: https://github.com/jadedjelly/M8-Jenkins-demos.git
    - use the correct credentials
- The Script path, is set to the newly created Jenkinsfile in the repo
    - This contains the groovy script that configires the pipeline
- click save
- the Job fails because the Jenkisnfile doesn't exist

- we create a simple Jenkinsfile on the repo, with the below:
```groovy
pipeline {

    agent any

    stages {

        stage("build") {

            steps {
              echo "building the application..."

            }
        }
        stage("test") {

            steps {
              echo "testing the application..."
            }
        }
        stage("deploy") {

            steps {
              echo "deploying the application..."
            }
        }
    }
}
```
- The build is a success and we can see the output below:

![M8image01.png](assets/M8image01.png)

- There's also a new output, that shows each stage that we defined (build, test, deploy) & this will show if it failed or not

![M8image02.png](assets/M8image02.png)

# Create a complete pipeline

- starting off we are using a basic Jenkinsfile as below:

```groovy
pipeline {
    agent any
    stages {
        stage('build') {
            steps {
                script {
                    echo "Building the application..."
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    echo "Deploying the application..."
                }
            }
        }
    }
}

```

NOTES:
- *Note: We are building a Jenkinsfile to replicate what we did in the Freestyle build, that being, using docker build, login and push*
- Rememeber if you need to use sh commands for maven you need to define them in tools
- To use credentials in this case docker, we use "withCredentials", we are extracting the username and passwords part of the credentialsId variable, we then create a block adding the {}, inside we have access to those tow variables
    - dont forget, if your using a repo other than DOckerHub, you would define this on the line with the credentials 
- What the new code now looks like is below:

```groovy
pipeline {
    agent any
    tools {
      maven 'maven-3.9'
    }
    stages {
        stage('build jar') {
            steps {
                script {
                    echo "Building the application..."
                  sh 'mvn package'
                }
            }
        }
      stage('build image') {
            steps {
                script {
                    echo "Building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'Docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh 'docker build -t jadedjelly/mod8-jenkins:jma-2.0 .'
                        sh 'echo $PASS | docker login -u $USER --password-stdin'
                        sh 'docker push jadedjelly/mod8-jenkins:jma-2.0'
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    echo "Deploying the application..."
                }
            }
        }
    }
}
```

- We get a success, and running through the logs we can see it checkedout the scm, then installed the tools, built the jar file, built the docker image, then deployed to my private dockerhub repo

![M8image03.png](assets/M8image03.png)


and below is the private docker repo:

![M8image04.png](assets/M8image04.png)

- We can also run it with a script.groovy to make it neater, as below:

```groovy
def buildJar() {
    echo "building the application..."
    sh 'mvn package'
} 

def buildImage() {
    echo "building the docker image..."
    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
        sh 'docker build -t jadedjelly/mod8-jenkins:jma-2.0 .'
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh 'docker push jadedjelly/mod8-jenkins:jma-2.0'
    }
} 

def deployApp() {
    echo 'deploying the application...'
} 

return this

```

- And below is the updated Jenkinsfile that uses said script:

```groovy
def gv

pipeline {
    agent any
    tools {
      maven 'maven-3.9'
    }
    stages {
        stage("init") {
            steps {
                script {
                    gv = load "script.groovy"
                }
            }
        }
        stage('build jar') {
            steps {
                script {
                    gv.buildJar()
                }
            }
        }
      stage('build image') {
            steps {
                script {
                    gv.buildImage()
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    gv.deployApp()
                }
            }
        }
    }
}

```

- This makes the Jenkinsfile much easier to read, with all the logic being refrenced from the script
- This has run successfully, using the script.groovy file

