example to create ReactJs
ref: <https://reactjs.org/docs/create-a-new-react-app.html>

### Open existing project
> Any Static Web Application. Here example use React-Js

### Setup Sam and deploy
Create template.yaml at root project
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Serverless website

Resources:
  CloudFrontOriginAccessIdentity:
    Type: 'AWS::CloudFront::CloudFrontOriginAccessIdentity'
    Properties:
      CloudFrontOriginAccessIdentityConfig:
        Comment: 'Serverless website OA'

  CloudfrontDistribution:
    Type: "AWS::CloudFront::Distribution"
    Properties:
      DistributionConfig:
        Comment: "Cloudfront distribution for serverless website"
        DefaultRootObject: "index.html"
        Enabled: true
        HttpVersion: http2
        # List of origins that Cloudfront will connect to
        Origins:
          - Id: s3-website
            DomainName: !GetAtt S3Bucket.DomainName
            S3OriginConfig:
              # Restricting Bucket access through an origin access identity
              OriginAccessIdentity: 
                Fn::Sub: 'origin-access-identity/cloudfront/${CloudFrontOriginAccessIdentity}'
        # To connect the CDN to the origins you need to specify behaviours
        DefaultCacheBehavior:
          # Compress resources automatically ( gzip )
          Compress: 'true'
          AllowedMethods:
            - GET
            - HEAD
            - OPTIONS
          ForwardedValues:
            QueryString: false
          TargetOriginId: s3-website
          ViewerProtocolPolicy : redirect-to-https
        CustomErrorResponses:
          - ErrorCode: '403'
            ResponsePagePath: "/index.html"
            ResponseCode: '200'
            ErrorCachingMinTTL: '10'
          - ErrorCode: '404'
            ResponsePagePath: "/index.html"
            ResponseCode: '200'
            ErrorCachingMinTTL: '10'


  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      # Change bucket name to reflect your website
      BucketName: <BUCKET NAME>

  S3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref S3Bucket
      PolicyDocument:
      # Restricting access to cloudfront only.
        Statement:
          -
            Effect: Allow
            Action: 's3:GetObject'
            Resource:
              - !Sub "arn:aws:s3:::${S3Bucket}/*"
            Principal:
              AWS: !Sub "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${CloudFrontOriginAccessIdentity}"

Outputs:
  CloudfrontDistributionID:
    Description: Pass Variable Cloudfront Distribution ID
    Value: !Ref CloudfrontDistribution
    # Export:
    #   Name: <EXPORT NAME>
  S3BucketName:
    Description: Pass Variable S3 Bucket DomainName
    Value: !Ref S3Bucket
    # Export:
    #   Name: <EXPORT NAME>
```

### Build and Deploy using CLI

```sh
sam build
```
First time deploy use flat `--guided` for setup sam config
```sh
sam deploy --guided
```

### Upload web static application to S3
CLI upload file
```sh
aws s3 cp build s3://S3_BUCKET --acl public-read --recursive
```
CLI 
```sh
aws cloudfront create-invalidation  --paths "/*" --distribution-id CLOUDFRONT_ID
```

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
        EnvironmentVariables:
          - Name: S3_BUCKET
            Value: !ImportValue <EXPORT NAME>
            Type: PLAINTEXT
          - Name: CLOUDFRONT_ID
            Value: !ImportValue <EXPORT NAME>
            Type: PLAINTEXT
      Source:
        Location: <GIT REPO PATH>
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
        - arn:aws:iam::aws:policy/CloudFrontFullAccess
```
Next

```sh
sam build
sam deploy --guided 
```

### Create Build Specification

ref: <https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html>

create file `buildspec.yaml` root at my-app

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: 12
  pre_build:
    commands:
      - npm install -g yarn
      - echo Installing source NPM dependencies...
      - yarn install
  build:
    commands:
      - yarn build
      - sam build 
      - sam deploy --no-fail-on-empty-changeset
      - aws s3 cp build s3://$S3_BUCKET --acl public-read  --recursive
      - aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_ID --paths "/*"
```
Before push you need to update file .gitignore
add
```
.aws-sam
```
next
```
git add .
git commit
git push
```