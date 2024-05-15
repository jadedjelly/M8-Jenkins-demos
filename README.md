## Demo Project: 
### Dynamically Increment Application version in Jenkins Pipeline

#### Technologies used:
- Jenkins, Docker, GitLab, Git, Java, Maven

#### Project Description:
1. Configure CI step: Increment patch version
2. Configure CI step: Build Java application and clean old artifacts
3. Configure CI step: Build Image with dynamic Docker Image Tag
4. Configure CI step: Push Image to private DockerHub repository
5. Configure CI step: Commit version update of Jenkins back to Git repository
6. Configure Jenkins pipeline to not trigger automatically on CI build commit to avoid commit loop

- Open intllij to the Java-maven-app & cd to the same folder in terminal
    - Run the below command to increment version, build-helper finds the version tag inside the pom file, and parses the data into 3 parts (major.minor.patch)
    - versions:set - sets the version between version text, have to set a new version we want to set in its place, add it as a param
    - since the current version needs to increment from the old one, we need this to be setup if its a major / minor / patch, the parse-version, also calculates the next increment for each part
    - This is done via variables (..=\${...})
        - the 1st 2x are variables we dont want to change, on the 3rd this will change notice the "nextIncrementalVersion"
        - the last bit "versions:commit" this will remove the old copy of the pom and we'll have the new one


# Increment Versions locally - Java Maven Project

```bash
## plugin (build-helper) available out of the box
mvn build-helper:parse-version versions:set -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion} versions:commit
```

![M8image01.png](assets/M8image01.png)

**Note: the / between the variables added \\1.\\1.\\1 as a version, on removal this now shows correctly**

```bash
mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit
```

![M8image02.png](assets/M8image02.png)

Nana runs it again, to show the incremental change
She goes on to describe a scenario where by a Dev is refactoring an apps code, and this changes so much that the company release it as a minor patch change (rather than a patch as we previously did)

The line we used previous would be changed as below:
```bash
mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.nextMinorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit
```

and we can see the change below:

![M8image03.png](assets/M8image03.png)

**NOTE: stupidly left "nextIncrementalVersion" instead of changing it to "incrementalVersion"**

Every build tool shares the same concept for versioning, you can download plugins that can also do it

# Increment Application version on Jenkins - Java Maven Project

![M8image04.png](assets/M8image04.png)

- Open the Jenkinsfile inside java-maven-app, and we input the below:

**NOTE: we escape out of the $ slightly different here, using the '' & \\\ - these escapes didnt work last time...**
**NOTE2: =~ is used in groovy to "find" **
**NOTE3: **

```groovy
def gv

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'Docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "docker build -t jadedjelly/mod8-jenkins:${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin'
                        sh "docker push jadedjelly/mod8-jenkins:${IMAGE_NAME}"
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    echo 'deploying docker image...'
                }
            }
        }
     }
}
}

```

And then we need to adjust the Dockerfile, to be more dynamic rather than having the hardcoded version

**NOTE: In the Dockerfile, we had to replace ENTRYPOINT with CMD, due to the change in the version name**

```bash
FROM amazoncorretto:8-alpine3.17-jre

EXPOSE 8080

COPY ./target/java-maven-app-*.jar /usr/app/
WORKDIR /usr/app

CMD java -jar java-maven-app-*.jar

```

We have to make another change in the Jenkinsfile, this is because a new jar file will be created, and the Jenkinsfile is looking at a wildcard, we do this by executing mvn clean

```groovy
        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
```

**Note2: Needed to change git repo, on MB, set to gitlab webhook demo**

- successfuly run! - though had issues with IDE not showing errors....
**Note to self: Review settings for Intellij, error handling, not showing correctly for Groovy, specifically Jenkinsfile**

![M8image05.png](assets/M8image05.png)

![M8image06.png](assets/M8image06.png)


# Commit version update of Jenkins back to Git repository

- we're adding a new stage, after deploy
    - 1st we need creds to access the repo
      - Using the "withCredentials" command (used for the Docker repo), chaning the CredentialsId to our github repo, we can extract the user & pass
          - USER & PASS are unique between the blocks
      - then we can run the usual git commands (add, commit & push)
        - with the push we are pushing specifically to the Jenkins-jobs branch (hence the HEAD:jenkins-jobs)

**NOTE: HEAD refers to the currently checked-out branch's latest commit**

- Before we can execute, we need to tell Jenkins what origin is, so we add the "git remote" command (above the git add) and setting the origin url

```bash
sh "git remote set-url origin https://${USER}:${PASS}@github.com/jadedjelly/Jenkins_Module8.git"
sh 'git add .'
sh 'git commit -m "ci: version bump to pom.xml'
sh 'git push origin HEAD:jenkins-jobs'
```

- we also print out some useful information for git, via

```bash
sh 'git status'
sh 'git branch'
sh 'git config --list'
```

- we also need to set git configurations, we can use the --global switch, or we can do without by running the "user.email" switch - but if we want this for all projects we would set it for global, without it it's just for this project
    - Note, we need to set a user email, otherwise git compalins about this (metadata during commits)

**Alternativly, we could ssh to Jenkins server and set the config**

```bash
sh 'git config --global user.email "jenkins@example.com"'
sh 'git config --global user.name "jenkins"'
```

- the above only needs to be set once, so doing it via the server would be easier, as we'll have to remove these 2 lines in a future job

- The entire block is below:

```bash
stage('commit version update'){
            steps{
                script{
                    withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'

                        sh 'git status'
                        sh 'git branch'
                        sh 'git config --list'

                        sh "git remote set-url origin https://${USER}:${PASS}@github.com/jadedjelly/Jenkins_Module8.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump to pom.xml'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
```

- Running a Build job for Jenkins-jobs

**Note2: Multiple issues exp'd error: "insecure interpolation of sensitive variables" - FIX: Use Personal access tokens!**

![M8image07.png](assets/M8image07.png)

![M8image08.png](assets/M8image08.png)

- Note in the console output, Jenkins isn't checking out the branch, it checks out the last commit hash

## Ignore Jenkins Commit for Jenkins Pipeline Trigger

- This is done via a plugin, go to plugins and search for
    - [Ignore Committer Strategy](https://plugins.jenkins.io/ignore-committer-strategy/)
    - install

![M8image09.png](assets/M8image09.png)

- after installation, head back to the Multibranch pipeline > click configure
    - Inside the Branch Sources, there's a new option called "Build strategies", click add and an option is available "ignore Commiter strategy" as below:

![M8image10.png](assets/M8image10.png)

In the text field we list authors by email address (recall we used the "Jenkins" user for the servers git config), so here we can just pop in "jenkins@example.com"

![M8image11.png](assets/M8image11.png)

- after the commit, the build ran, but did not trigger another build (woot!)

Doing it via the users email is more "elegant" than using a keyword / string search within the code

---------------------------------------------------------------------------------------------------
