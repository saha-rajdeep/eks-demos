---
AWSTemplateFormatVersion: 2010-09-09


Parameters:

  KubectlRoleName:
    Type: String
    Default: EksWorkshopCodeBuildKubectlRole
    Description: IAM role used by kubectl to interact with EKS cluster
    MinLength: 3
    MaxLength: 100
    ConstraintDescription: You must enter a kubectl IAM role


Resources:


  CodePipelineServiceRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: codepipeline-access
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Resource: "*"
                Effect: Allow
                Action:
                  - codebuild:StartBuild
                  - codebuild:BatchGetBuilds
                  - codecommit:GetBranch
                  - codecommit:GetCommit
                  - codecommit:UploadArchive
                  - codecommit:GetUploadArchiveStatus
                  - codecommit:CancelUploadArchive
                  - iam:PassRole
              - Resource: "*"
                Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:GetObject
                  - s3:GetObjectVersion
                  - s3:GetBucketVersioning


  CodeBuildServiceRole:
    Type: AWS::IAM::Role
    Properties:
      Path: /
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: root
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Resource: !Sub arn:aws:iam::${AWS::AccountId}:role/${KubectlRoleName}
                Effect: Allow
                Action:
                  - sts:AssumeRole
              - Resource: '*'
                Effect: Allow
                Action:
                  - eks:Describe*
              - Resource: '*'
                Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
              - Resource: '*'
                Effect: Allow
                Action:
                  - ecr:GetAuthorizationToken
              - Resource: '*'
                Effect: Allow
                Action:
                  - ec2:CreateNetworkInterface
                  - ec2:DescribeDhcpOptions
                  - ec2:DescribeNetworkInterfaces
                  - ec2:DeleteNetworkInterface
                  - ec2:DescribeSubnets
                  - ec2:DescribeSecurityGroups
                  - ec2:DescribeVpcs
                  - ec2:CreateNetworkInterfacePermission
              - Resource: "*"
                Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                  - s3:GetObjectVersion
              - Resource: '*'
                Effect: Allow
                Action:
                  - ecr:GetDownloadUrlForLayer
                  - ecr:BatchGetImage
                  - ecr:BatchCheckLayerAvailability
                  - ecr:PutImage
                  - ecr:InitiateLayerUpload
                  - ecr:UploadLayerPart
                  - ecr:CompleteLayerUpload
