
## Lab 2 - Automate deployment for testing

### Stage 1: Prepare environment for Testing

1. Run the CloudFormation stack using the following AWS CLI command:

```console
user:~/environment/WebAppRepo (master) $ aws cloudformation create-stack --stack-name DevopsWorkshop-Env \
--template-body https://github.com/awslabs/aws-devops-essential/raw/master/templates/02-aws-devops-workshop-environment-setup.template \
--capabilities CAPABILITY_IAM
```

**_Note_**
  - The Stack will have a VPC w/ 1 public subnet, an IGW, route tables, ACL, 2 EC2 instances. Also, the EC2 instances will be launched with a User Data script to **automatically install the AWS CodeDeploy agent**.
  - **Verify** that by visiting the **EC2 Console** and view option for **user data**.You would see the following script.

```console
#!/bin/bash -ex
yum install -y aws-cli
cd /home/ec2-user/
wget https://aws-codedeploy-us-east-1.s3.amazonaws.com/latest/codedeploy-agent.noarch.rpm
yum -y install codedeploy-agent.noarch.rpm
service codedeploy-agent start
```
  - **Verify** in **System Log** by clicking Instance Settings->Get System Log for below entries
  
  ```shell
Check if there is a codedeployagent config file.
Start codedeploy-agent in post hook if this is a first install.
Installing codedeploy-agent auto-update cron in '/etc/cron.d/codedeploy-agent-update'...
Installing codedeploy-agent auto-update cron in '/etc/cron.d/codedeploy-agent-update'...Complete
  Verifying  : codedeploy-agent-1.0-1.1458.noarch                           1/1 

Installed:
  codedeploy-agent.noarch 0:1.0-1.1458                                          

Complete!
+ service codedeploy-agent start
  ```
  
  - You can refer to [this instruction](http://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install.html) to install the CodeDeploy agent for other OSs like Amazon Linux, RHEL, Ubuntu, or Windows.
  - AWS CodeDeploy can deploy to both Amazon EC2 instances and on-premises instances.To know more [visit](http://docs.aws.amazon.com/codedeploy/latest/userguide/instances.html).

***

### Stage 2: Create CodeDeploy Application and Deployment group

1. Run the following to create an application for CodeDeploy.

```console
user:~/environment/WebAppRepo (master) $ aws deploy create-application --application-name DevOps-WebApp
```

2. Run the following to create a deployment group and associates it with the specified application and the user's AWS account. You need to replace the service role with **DeployRoleArn Value** we created using roles CFN stack.

```console
user:~/environment/WebAppRepo (master) $ aws deploy create-deployment-group --application-name DevOps-WebApp \
--deployment-config-name CodeDeployDefault.OneAtATime \
--deployment-group-name DevOps-WebApp-BetaGroup \
--ec2-tag-filters Key=Name,Value=DevWebApp01,Type=KEY_AND_VALUE \
--service-role-arn <<REPLACE-WITH-YOUR-CODEDEPLOY-ROLE-ARN>>
```

**_Note:_** We are using the tags to attach instances to the deployment group.

3. Let us review all the changes by visiting the CodeDeploy Console.

***

### Stage 3: Prepare application for deployment

1. Without an AppSpec file, AWS CodeDeploy cannot map the source files in your application revision to their destinations or run scripts at various stages of the deployment.
2. Copy the template into a text editor and save the file as **_appspec.yml_** in the **_WebAppRepo_** directory of the revision.

```yml
version: 0.0
os: linux
files:
  - source: /target/javawebdemo.war
    destination: /tmp/codedeploy-deployment-staging-area/
  - source: /scripts/configure_http_port.xsl
    destination: /tmp/codedeploy-deployment-staging-area/
hooks:
  ApplicationStop:
    - location: scripts/stop_application
      timeout: 300
  BeforeInstall:
    - location: scripts/install_dependencies
      timeout: 300
  ApplicationStart:
    - location: scripts/write_codedeploy_config.sh
    - location: scripts/start_application
      timeout: 300
  ValidateService:
    - location: scripts/basic_health_check.sh

```

As a sample shown below:

![appspec](./img/app-spec.png)

3. **_Review_** the **_script folder_** in the repo for the various scripts like Start, Stop, health check etc. These scripts will be called as per the hook definition in **_appspec.yml_** file during deployment.
4. Since we are going to deploy the application via CodeDeploy, we need to package additional files needed by CodeDeploy. Let us **_make change_** to the **_buildspec.yml_** to incorporate the changes.

```yml
version: 0.1

phases:
  install:
    commands:
      - echo Nothing to do in the install phase...
  pre_build:
    commands:
      - echo Nothing to do in the pre_build phase...
  build:
    commands:
      - echo Build started on `date`
      - mvn install
  post_build:
    commands:
      - echo Build completed on `date`
artifacts:
  files:
    - appspec.yml
    - scripts/**/*
    - target/javawebdemo.war

```

5. Save the changes to buildspec.yml. Run **_git add, commit, and push_** the changes to CodeCommit repo.

***

### Stage 4: Deploy an application revision

1. Run the **_start-build_** command:

```console
user:~/environment/WebAppRepo (master) $ aws codebuild start-build --project-name devops-webapp-project
```

2. Visit the CodeBuild Console to ensure build is successful. Upon successful completion of build, we should see new **_WebAppOutputArtifact.zip_** upload to the configured CodeBuild S3 Bucket.
3. Get the **_eTag_** for the object **WebAppOutputArtifact.zip** uploaded to S3 bucket. You can get etag by visiting S3 console. Or, executing the following command.

```console
user:~/environment/WebAppRepo (master) $ aws s3api head-object --bucket <<YOUR-CODEBUILD-OUTPUT-BUCKET>> \
--key WebAppOutputArtifact.zip

```

As a sample S3 properties console showing etag below:

![etag](./img/etag.png)

4. Run the following to create a deployment. **_Replace_** <<YOUR-CODEBUILD-OUTPUT-BUCKET>> with your **_S3 bucket name_** created in Lab 1. Also, update the **_eTag_** based on previous step.

```console
user:~/environment/WebAppRepo (master) $ aws deploy create-deployment --application-name DevOps-WebApp \
--deployment-group-name DevOps-WebApp-BetaGroup \
--description "My very first deployment" \
--s3-location bucket=<<YOUR-CODEBUILD-OUTPUT-BUCKET>>,key=WebAppOutputArtifact.zip,bundleType=zip,eTag=<<YOUR-ETAG-VALUE>>
```

5. **Confirm** via IAM Roles, if associated EC2 instance has appropriate permissions to read from bucket specified above. If not, you will get Access Denied at the DownloadBundle step during deployment.

6. **Verify** the deployment status by visiting the **CodeDeploy console**.
7. Check the deploy console for status. if the deployment failed, then look at the error message and correct the deployment issue.
8. if the status of deployment is success, we should be able to view the web application deployed successfully to the EC2 server namely **_DevWebApp01_**
9. Go to the **EC2 Console**, get the **public DNS name** of the server and open the url in a browser. You should see a sample web application.

### Summary

This **concludes Lab 2**. In this lab, we successfully created CodeDeploy application and deployment group. We also modified buildspec.yml to include additional components needed for deployment. We also successfully completed deployment of application to test server.You can now move to the next Lab,

[Lab 3 - Setup CI/CD using AWS CodePipeline](3_Lab3.md)

**_✅ Do It Yourself (DIY):_** Environment variable for CodeBuild. How can we specifiy environment variable for your Build Environment. Environment variable contains a mapping of key/value scalars, where each mapping represents a single custom environment variable in plain text. key is the name of the custom environment variable, and value is that variable's value. To know more about how to pass environment variable. [Check the build spec reference](http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)

* Use SSM as Parameter Store.
* If you want to store environment variables outside, then use Amazon EC2 Systems Manager Parameter Store. [Check this for how to store parameter and secure them using KMS](http://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-walk.html#sysman-paramstore-console).
* To allow AWS CodeBuild to retrieve custom environment variables stored in Amazon EC2 Systems Manager Parameter Store, you must add the ssm:GetParameters action to your AWS CodeBuild service role.

***
