# Project 8: Continues integration on AWS
In this project, I used AWS to build a Continues Integration Pipeline, configured CodeCommit as a private repository to store our code, I used Visual Studio as the local coding IDE, configured it to connect with COdeCommit via ssh. When there is a new commit, cloudwatch triggers the Pipeline, first the code is tested in Sonarcloud for bugs, if the threshold is not met, the pipeline notifies failure and stops, else the pipeline proceeds to build. The build artifact is then deployed to an S3 bucket, a notification(SNS) is sent to the configured team via email. This project reduces Operational overhead, has a very short Time To Repair, needs no human intervention and fault isolation.

## Services Used
- AWS Code Commit
- AWS Code Artifact
- AWS Code Build
- AWS Code Deploy
- AWS Code Pipeline
- S3
- CloudWatch
- IAM
- Sonar Cloud

## Steps
1. Code Commit Configurations
In codecommit, we create a repository
```
vprofile-code-repo
```
Creat an IAM user
```
vprofile-cart-admin
```
Create a policy to access all CodeArtifact services, but limit the resource to the repository we just created _vprofile-code-repo_ and attach this policy to the user _vprofile-cart-admin_

- Configure SSH
  On our command line, we run _ssh-keygen_ to generate a new ssh key, We give it the name
  ```
    vpro-codecommit_rsa
  ```
  We then cat the public key that has been created copy it and return to our AWS console.
  Again select the user created above, click security credentials and scroll down to SSH keys for AWS CodeCommit, and click. Paste it in the window that is waiting the key and click on save.
- Create a config file
  On our command line, cd into the .ssh file and create a config file with the following:
  ```
  Host git-codecommit.*.amazonaws.com
  User APK***************
  IdentityFile ~/.ssh/vpro-codecommit_rsa

  ```
  Where APK*************** is copied from the SSH key Id on the AWS Console, security credentials, the IndentityFile is the path to the private ssh key we created earlier. The config file permission has to be 600, so we cd into the containing folder and run
  ```
  chmod 600 config
  ```
  To test our connection, we run
  ```
  ssh git-codecommit.us-east-2.amazonaws.com
  ```
  We will get a confirmation to have successfully authenticated to codecommit.

- Storing our code on Codecommit
  We clone the project repository using
  ```
  git clone [https://](https://github.com/devopshydclub/vprofile-project.git)https://github.com/devopshydclub/vprofile-project.git
  ```
  CD into the project folder _vprofile-project_ and checkout of the branches by running the following code.
  ```
  git branch -a | grep -v HEAD | cut -d '/' -f3 | grep -v master > /tmp/branches
  for i in `cat /tmp/branches` ; do git checkout $i;done;
  ```
  Change remote url to our codecommit url and push our code to code commit
  ```
  git remote rm origin
  git remote add origin ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/vprofile-code-repo
  git push origin --all
  ```
  And verify our code is there

  ![](successful upload on codecommit)

2. AWS CodeArtifact token
   On the console, search codeartifact and open, create a new repository _vprofile-maven-repo_. For public upstream repositories, we select _maven-central-store_ Click on next, for domain -> this AWS account, then we give the name "visualpath" then next, confirm the settings and click on create. We should see two created Repositories:
  ![](create-repository-codeArtifact)
 
  Next we click on maven-central-store and click connection instructions.
- Connection
  On AWS console, we create a new user with programmatic access, attach AWScodeArtifactAdmin policy and download the csv file. On awscli on local machine, run
  ```
  aws configure
  ```
  enter the just downloaded credentials(Access keys and secret access keys) then run the command
  ```
  export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain visualpath --domain-owner 997450571655 --region us-east-1 --query authorizationToken --output text`
  ```
  To generate a codeArtifact token, to see it, run
  ```
  echo $CODEARTIFACT_AUTH_TOKEN
  ```
  It will be needed later on

  - Updating settings.xml and pom.xml files
    Checkout to ci-aws on command line, open the file aws-files/settings.xml change the following lines having the urls and put in your aws account code:
    settings.xml
```
<profiles>
  <profile>
    <id>visualpath-maven-central-store</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <repositories>
      <repository>
        <id>visualpath-maven-central-store</id>
        <url>https://visualpath-99********.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
      </repository>
    </repositories>
  </profile>
</profiles>
```
And the mirrors
```
<mirrors>
  <mirror>
    <id>visualpath-maven-central-store</id>
    <name>visualpath-maven-central-store</name>
    <url>https://visualpath-997********.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
    <mirrorOf>*</mirrorOf>
  </mirror>
</mirrors>
```
For the pom.xml, we also change only the url
```
<mirrors>
  <mirror>
    <id>visualpath-maven-central-store</id>
    <name>visualpath-maven-central-store</name>
    <url>https://visualpath-9974505******.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
    <mirrorOf>*</mirrorOf>
  </mirror>
</mirrors>
```
- Sonarcloud setup
  Open sonar cloud via the link https://sonarcloud.io and signin with github.
  Create a token QR-icon top right corner -> my account, under security tab, type "vprofile-sonar-cloud" and generate token, the token should be stored in parameter stoke under sonartoken(secure-string). Now we click on the + (top right) -> analyze new project. We give the project name as vprofile-repo. Save the project-key to parameter store under Project.
  
- Configuring Parameter store
  Search parameter store, Create the following parameters
  ```
  HOST: https://sonarcloud.io
  Organization: <our organization key on sonarcloud>
  sonartoken: <generated sonar token>
  codeartifact_token: <the generated codeartifact token in commandline>
  Project: <project-key-from-sonarcloud>
  ```
  ![](parameter store)

3. Build Job in AWS Code build
   Our first build project will be to run code analysis on Sonarcloud, So in the console, search codebuild -> create build Project:
   
   Name: vprofile-Build
   source provider: AWS codecommit
   repository: vprofile-code-repo
   branch: ci-aws
   Operating system: Ubuntu
   runtime: standard
   image: aws/codebuild/standard:5.0
   environment type: Linux
   create a new service: role give it the name codebuild-v-service-role
   Insert build commands -> switch to editor and paste the following code
   
   ```
   version: 0.2
    env: 
      parameter-store:
        CODEARTIFACT_AUTH_TOKEN: codeartifact-token
    phases:
      install:
        runtime-versions:
          java: corretto8
        commands:
          - cp ./settings.xml /root/.m2/settings.xml
      pre_build:
        commands:
          - apt-get update
          - apt-get install -y jq 
          - wget https://dlcdn.apache.org/maven/maven-3/3.9.4/binaries/apache-maven-3.9.4-bin.tar.gz
          - tar xzvf apache-maven-3.9.4-bin.tar.gz
          - ln -s apache-maven-3.9.4 maven
      build:
        commands:
          - mvn clean install -DskipTests
    artifacts:
      files:
         - target/**/*.war
      discard-paths: yes
   ```
   
   Group name: vprofile-nvir-build
   stream name: sonarbuildjob
   
  
   Finally, click on create build project
   
   - Update codebuild-v-service-role in IAM to access parameter store
     Search the role used in creating the build project, attach policy and create new policy with the following inputs:
     service: system manager
     Actions -> Access level ->List(choose DescribeParameters)
     Actions -> Access level ->Read(chose DescribeDocumentParameters, GetParameters, GetParametersHistory, GetParameter, GetParametersByPath)
     Resources: *
     policy name: vprofile-sonarparametersaccess

   - Run Build
     ![](build-success)

  5. Build Artifact
     Create another build Job to build the artifact
     Name: vprofile-Build-artifact
     source provider: AWS codecommit
     repository: vprofile-code-repo
     branch: ci-aws
     runtime: standard
     image: aws/codebuild/standard:5.0
     environment type: Linux
     create a new service: role give it the name codebuild-vprofile-Build-Artifact-service-role
     Insert build commands -> switch to editor and paste the following code
     ```
     version: 0.2
      env: 
        parameter-store:
          CODEARTIFACT_AUTH_TOKEN: codeartifact-token
      phases:
        install:
          runtime-versions:
            java: corretto8
          commands:
            - cp ./settings.xml /root/.m2/settings.xml
        pre_build:
          commands:
            - apt-get update
            - apt-get install -y jq 
            - wget https://dlcdn.apache.org/maven/maven-3/3.9.4/binaries/apache-maven-3.9.4-bin.tar.gz
            - tar xzvf apache-maven-3.9.4-bin.tar.gz
            - ln -s apache-maven-3.9.4 maven
        build:
          commands:
            - mvn clean install -DskipTests
      artifacts:
        files:
           - target/**/*.war
        discard-paths: yes
      ```
      Group name: vprofile-nvir-build
      stream name: buildjob
   
      Create build job

     - Updating codebuild-vprofile-Build-Artifact-service-role
       In IAM, search the policy "codebuild-vprofile-Build-Artifact-service-role" and attach the policy(vprofile-sonarparametersaccess) to access parameter store.

     - Run Build Job
       In Codebuild, select vprofile-build-artifact and run it
       ![](build success artifact)

     5. Notifications and Pipeline
        - Create a notification Topic -> vprofile-pipeline-notification
        - Create a Subscription -> protocol=email
        - Confirm subscription, go to our email and click on confirm subscription in the mail from AWS SNS
        - Create pipeline:
          
               - goto pipeline and create pipeline
               - name: vprofile-CI-pipeline
               - Source provider: AWS codecommit
               - Repository name: vprofile-code-repo
               - branch name: ci-aws
               - detection options: cloudwatch events
               - build provider: AWS codebuild
               - project name: vprofile-Build-artifact and skip deploy.
               - Stop pipeline execution and edit it
           
       - Add Test stage:
         
              - Add actions group(after the first stage)
              - Actions name: sonar-code-analysis
              - Action provider: AWS COdeBuild
              - Input artifact: SourceArtifact
              - Project name: vprofile-Build -> done
       - Add Deploy stage:
         
             - Create s3 buck: vprofile-build-artifact12, create a folder 'pipeline-artifacts'
             - Add actions group
             - Actions name: Deploy-to-S3
             - Action provider: Amazon s3
             - Input artifact: BuildArtifact
             - Bucket: vprofile-build-artifact12
             - S3 object key: pipeline-artifacts
             - extract file before deploy
             - Done, save pipeline
       - Set notifications

             - goto pipeline settings -> vprofile-CI-Pipeline-Notification
             - Events that trigger notifications: Select all
             - Configured Targets: SNS Topic, choose target: select SNS notification topic created earlier
       
        - Run Pipeline
          Click Release change to test the pipeline
          ![](pipeline)

          Check the s3 bucket
          ![](s3 bucket)

     

















          
          
     
     

