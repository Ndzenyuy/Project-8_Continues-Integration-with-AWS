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
  Where APK*************** is copied from the user
Click on connections steps for ssh
