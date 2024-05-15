## Demo Project: 
### Create a Jenkins Shared Library

#### Technologies used:
- Jenkins, Groovy, Docker, Git, Java, Maven

#### Project Description:
Create a Jenkins Shared Library to extract common build logic:
1. Create separate Git repository for Jenkins Shared Library (JSL) project
2. Create functions in the JSL to use in the Jenkins pipeline
3. Integrate and use the JSL in Jenkins Pipeline (globally and for a specific project in Jenkinsfile)

- We create a new groovy project in intillij
- Add a folder called var
- add a new file called "buildJar.groovy"
    - Inside the Jenkinsfile you'll reference the name of the file
- inside the new file we define the logic, as below for buildJar
- it's also best practice to add the #! for groovy to the file
    - #!/user/bin/env groovy

the code used is exactly the same as we wrote for the script.groovy

```groovy
#!/user/bin/env groovy

def call() {
    echo "building the application..."
    sh 'mvn package'
}
```
- we do the same for buildImage

- Now we create & push to the repo on Github
    - init, commit, push etc

## Make shared Library globally available

If the SL is used by all teams, it makes sense to make it available globally

- Head to Manage Jenkins
    - system
        - scroll down to Global Pipeline Libraries & click add
            - name: Jenkins-shared-library
            - Default version: here we can use a commit hash, or master, or etc. You should be versioning the JSL. we set it as **main**, obviously if something breaks here it breaks the pipelines
            - I use github, so fill in what is required (credentials & url)
            - click save
That's it, it's done. The SL is now available globally

## Use shared library in a Jenkinsfile

- create a new branch off main, and call it jenkins-shared-library

we keep the script.groovy as Nana has pointed out you might in the real world have references that use it

- we add the following to import that library as we defined in the global configuration, this is added under the shebang and above the def gv

**Note: if we didn't have the def gv definition we would need to put an underscore_ after the line as** 
*@Library('Jenkins-shared-library')_*

```groovy
@Library('Jenkins-shared-library')
```

- From the script.groovy we remove the buildJar & buildImage references and from the Jenkinsfile

```groovy
        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }
        stage("build image") {
            steps {
                script {
                    buildImage()
                }
            }
        }
```

- the calling of these 2x functions is so very simple buildJar() and buildImage()
- Run the scan Multibranch pipeline now
    - test was successful

![M8image01.png](assets/M8image01.png)

![M8image02.png(assets/M8image02.png)

## Using parameters in a shared Library

In the buildImage groovy file, we have hardcoded the image name & tag, but this is impractical, as we would want to pass the image as a parameter, and each build would create a new repo and tag
- inside the buildImage.groovy file, we add a parameter to the call (similar in most languages)

```groovy
def call(String imageName) {
    echo "building the docker image..."
    withCredentials([usernamePassword(credentialsId: 'Docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
        sh "docker build -t $imageName ."
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh "docker push $imageName"
    }
}

```

- In an earlier build, we had logic which was being built using EnvVariables. We add this to the buildJar.groovy file as below

```groovy
def call() {
    echo "building the application for branch $BRANCH_NAME"
    sh 'mvn package'
}
```

- Back in the Jenkinsfile, we add the below to the buildImage

```groovy
        stage("build image") {
            steps {
                script {
                    buildImage 'jadedjelly/mod8-jenkins:jma-3.0'
                }
            }
        }
```

- We push the changes and run the scan again
- viewing the build logs, we can see the params that are now displayed within the logs, and checking dockerhub we can see the new image

![M8image03.png(assets/M8image03.png)


## Extract logic to Groovy Classes

- Create a new package inside the src folder, call it "com.example"
- then create a new file inside the com.example package, call it "Docker.groovy"

*Note: We don't have the ability to use Env Variables / withCredentials, etc as we would in the scripts located in vars folder*

- We pass parameters from these methods and we call it
- the line below, will hold all the info, inc enviroments (Docker(script) & the commands and methods for the pipeline

```groovy
package com.example

class Docker implements Serializable{

    Docker(script){
        
    }
}
```

- "this.script = script" will allow us to execute all those commands
```groovy
    def buildDockerImage(string imageName) {
        echo "building the docker image..."
        withCredentials([usernamePassword(credentialsId: 'Docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
            sh "docker build -t $imageName ."
            sh 'echo $PASS | docker login -u $USER --password-stdin'
            sh "docker push $imageName"
        }
    }
}
```

- Nana points out you can have multiple functions here, each one executing a single command
- Since groovy doesn't have access / cant resolve the commands used inside the block and we have defined it inside the def script, we add "script." so they can be, as below:

```groovy
def buildDockerImage(string imageName) {
        script.echo "building the docker image..."
        script.withCredentials([script.usernamePassword(credentialsId: 'Docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
            script.sh "docker build -t $imageName ."
            script.sh 'echo $PASS | docker login -u $USER --password-stdin'
            script.sh "docker push $imageName"
        }
```

- we need to edit the varibales used for authentication, as leaving it as it is will be used as a string rather than what it is

```groovy
...
            script.sh "echo '${script.PASS}' | docker login -u '${script.USER}' --password-stdin"
...
```

- we could do the same if we wanted to use $BRANCH_NAME

The function is ready, now it's time to edit the buildImage.groovy

- as always we need to import the new made package, using

```groovy
import com.example.docker
```

- inside the Jenkinsfile we don't need to change the syntax for buildImage, as we are still referring to it!
- This is a best practice, and makes it easier to use
- pushed to repo
- and we retest pipeline

*Note2: Nana' syntax didn't work for me and had to change some variables names...*

- Run Build now, and got a successful build, see below:

![M8image04.png(assets/M8image04.png)

## split "buildDockerImage" into seperate steps

Using the newly created Docker.groovy file, we're going to split it down further, creating 3 fucntions

- We seperate each part of the file, build, login and push, as below:

```groovy
#!/user/bin/env groovy
package com.example

class Docker implements Serializable {

    def script

    Docker(script) {
        this.script = script

    }

    def buildDockerImage(String imageName) {
        script.echo "building the docker image..."
            script.sh "docker build -t $imageName ."
        }

    def dockerLogin(){
        script.withCredentials([script.usernamePassword(credentialsId: 'Docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
            script.sh "echo '${script.PASS}' | docker login -u '${script.USER}' --password-stdin"
        }
    }

    def dockerPush(String imageName){
        script.sh "docker push $imageName"
    }
}
```

- Just as before we can ref the fucntions in our scripts, we also create another 2x files in vars
    - dockerLogin.groovy
    - dockerPush.groovy

dockerLogin.groovy
```groovy
#!/user/bin/env groovy
import com.example.Docker
def call() {
    return new Docker(this).dockerLogin()
}
```

dockerPush.groovy
```groovy
#!/user/bin/env groovy
import com.example.Docker
def call(String imageName) {
    return new Docker(this).dockerPush(imageName)
}
```

- inside the Jenkinsfile, we can call the functions as you would
*Note: I have changed the version to 4.0*

```groovy
...
        stage("build and push image") {
            steps {
                script {
                    buildImage 'jadedjelly/mod8-jenkins:jma-4.0'
                    dockerLogin()
                    dockerPush 'jadedjelly/mod8-jenkins:jma-4.0'
                }
            }
        }
...
```

- pushed the updates and running Build now
- success

![M8image05.png(assets/M8image05.png)

![M8image06.png(assets/M8image06.png)

## Project scoped Shared Library

- 1st we remove the library from the global area (system > scroll down to SL and click delete and save)
- In the Jenkinsfile, we remove the @Library... as it obvs won't work, instead
    - we add the below, note, the same as we had to use a version, the same is done here after the @ (main is master for me) - you can also add a tag, hash to this
    - we also add another attrib "retriever" which will call the modernSCM
        -inside we call the GitSCMSource,
        - remote, which is the git url
        - credentialsId - self explanatory

```groovy
#!/usr/bin/env groovy
library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
        [$class: 'GitSCMSource',
        remote: 'https://github.com/jadedjelly/jenkins-shared-library.git',
        credentialsId: 'github-creds'])
     

def gv
```

- pushed, and run and got success!

![M8image07.png(assets/M8image07.png)
