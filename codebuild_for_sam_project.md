### Create sam project
ref: <https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-getting-started-hello-world.html>

```sh
sam init
```

```sh
cd simple-app
```

### Run Local
ref: <https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-start-api.html>

```sh
sam local start-api
```
open link `http://127.0.0.1:3000/hello` at your browser.

Result
```
{"message":"hello world"}
```
### Build and Deploy using CLI

```sh
sam build
```
First time deploy use flat `--guided` for setup sam config
```sh
sam deploy --guided
```

Output
```sh
Stack Name [sam-app]: simple_app
AWS Region [us-east-1]: ap-southeast-1
#Shows you resources changes to be deployed and require a 'Y' to initiate deploy
Confirm changes before deploy [y/N]: N
#SAM needs permission to be able to create roles to connect to the resources in your template
Allow SAM CLI IAM role creation [Y/n]: Y
HelloWorldFunction may not have authorization defined, Is this okay? [y/N]: y
Save arguments to configuration file [Y/n]: Y
SAM configuration file [samconfig.toml]: 
SAM configuration environment [default]: 
```
Configuration will save to `samconfig.toml` and all template will deploy to AWS cloud. You also can monitor at AWS CloudFormation console

### Create github repository
create github repo and push

git init -> set remote -> push


## Setup CI/CD using AWS CodeBuild

### Set GitHub allow connect from AWS CodeBuild
GitHub ->  Organization

click setting at your organization name

go to Third-party access

click `your own authorized applications.` link

click AWS CodeBuild (Singapore)

and Grant your organization for access.

### AWS Connect to GitHub using OAuth
> note: to check connect list-source-credentials use cli > `aws codebuild list-source-credentials`

Go to AWS console -> CodeBuild

click Create Build Project

at Source

Source provider -> GitHub

Pick Connect using OAuth

click Connect to GitHub

Done Connect.


### Setup template for AWS CodeBuild

Create new directory and create file name `template.yaml`

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Template for creates Codebuild resources for a simple SAM Application

Parameters:
  CodeBuildName:
    Type: String
    Default: <PROJECT NAME>
    Description: Code Build Project Name
  GitBranchName:
    Type: String
    Default: <BRANCH NAME>
    Description: Git Branch Name

Resources:
  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Ref CodeBuildName
      Description: Build process for a CloudFormation deployed using AWS SAM
      ServiceRole: !GetAtt CodeBuildIAMRole.Arn
      Artifacts:
        Type: no_artifacts
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0
      Source:
        Location: <GIT REPOSITORY>
        Type: GITHUB
        GitCloneDepth: 1
      SourceVersion: !Ref GitBranchName
      Triggers:
        Webhook: true
        FilterGroups:
          - - Type: EVENT
              Pattern: PUSH
              ExcludeMatchedPattern: false
            - Type: HEAD_REF
              Pattern: !Ref GitBranchName
          - - Type: EVENT
              Pattern: PULL_REQUEST_MERGED
              ExcludeMatchedPattern: false
            - Type: HEAD_REF
              Pattern: !Ref GitBranchName
      BadgeEnabled: false
      LogsConfig:
        CloudWatchLogs: 
          Status: ENABLED
        S3Logs:
          Status: DISABLED
          EncryptionDisabled: false
      TimeoutInMinutes: 15
  CodeBuildIAMRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub 
        - 'role-${CODEBUILD}-full-access-${AWS::Region}'
        - { CODEBUILD: !Ref CodeBuildName }
      Description: Provides Codebuild permission to access API GW, Lambda and Cloudformation
      #Provide Codebuild permission to assume this role
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - "codebuild.amazonaws.com"
            Action:
              - "sts:AssumeRole"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AWSLambdaFullAccess
        - arn:aws:iam::aws:policy/AmazonAPIGatewayAdministrator
        - arn:aws:iam::aws:policy/AWSCloudFormationFullAccess
```
Next

```sh
sam build
sam deploy --guided 
```

### Create Build Specification

ref: <https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html>

create file `buildspec.yaml` root at simple-app

```yaml
version: 0.2
phases:
  build:
    commands:
      - sam build 
      - sam deploy --no-fail-on-empty-changeset
```

next 
```
git add .
git commit
git push
```

