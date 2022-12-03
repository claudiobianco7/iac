AWSTemplateFormatVersion: 2010-09-09
Description: >-
  Deploy a Cloud Front with S3 Bucket into Amazon Web Services (AWS) account for Dynamic Pinning.

######################
# Parameters section #
######################

Parameters:

  AccountOwnerEmail:
    Description: Specify a Account Owner Email.
    Type: String
    Default: elaine.santos@itau-unibanco.com.br

  AccountTechEmail:
    Description: Specify a Account Tech Email.
    Type: String
    Default: DDISTGERCANAISDIGITAISLATAM@correio.itau.com.br

  AccountTeamEmail:
    Description: Specify a Account Team Email.
    Type: String
    Default: DDISTGERCANAISDIGITAISLATAM@correio.itau.com.br

  CNAMES:
    Description: Specify a list of additional Custom Domain Names that you use separated with commas.
    Type: CommaDelimitedList
    Default: 'pinningcf.dev.aceleradoralatam.com,www.pinningcf.dev.aceleradoralatam.com'

  DEFAULTROOTOBJECT:
    Description: Specify only the default file name for the index document.
    Type: String
    Default: 'index.html'

  ORIGINPATH:
    Description: Specify the origin path (default folder) for CloudFront Distribution.
    Type: String
    Default: '/destination_path'

  LOGBUCKET:
    Description: Specify S3 Bucket name to store the access logs files.
    Type: String
    Default: 'itauopsdevlogs'
  
  CERTIFICATE:
    Description: Specify a ARN of valid ACM Certificate.
    Type: String
    Default: 'arn:aws:acm:us-east-1:335075401469:certificate/f5be6ec6-12d2-417a-95d4-4dd133529275'

  WAF:
    Description: Specify a ARN of valid WAF Web ACLs for Cloud Front.
    Type: String
    Default: 'arn:aws:wafv2:us-east-1:335075401469:global/webacl/FMManagedWebACLV2PublicACL1602872459764/67133653-4ce4-42ab-a0f0-eb710a63fe8f'

  BUCKETS3:
    Description: Specify S3 Bucket name where is ZIP file of Lambda function.
    Type: String
    Default: 'latam-create-lambada-from-zip-file'

  SUBNET:
    Description: Choose two Internal Subnets inside to attach this Lambda function.
    Type: 'List<AWS::EC2::Subnet::Id>'

  SECURITYGROUP:
    Description: You must specify a Security Group ID to this Lambda function.
    Type: AWS::EC2::SecurityGroup::Id

  LambdaVersion:
    Type: String    

####################
# Resource section #
####################

Resources:

  ## S3 Bucket to CDN
  s3BUCKET:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: Private
      BucketName: !Sub '${AWS::StackName}-s3-bucket-cdn'
      VersioningConfiguration:
        Status: Enabled
      LoggingConfiguration:
        DestinationBucketName: !Ref LOGBUCKET
        LogFilePrefix: '${AWS::StackName}-s3-bucket-cdn'    
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: True
        BlockPublicPolicy: True
        IgnorePublicAcls: True
        RestrictPublicBuckets: True
      Tags:
        - Key: StackName
          Value: !Ref AWS::StackName
        - Key: Region
          Value: !Ref AWS::Region
        - Key: Name
          Value: !Sub '${AWS::StackName}-s3-bucket'
        - Key: Owner
          Value: !Ref AWS::AccountId
        - Key: tech-team-email
          Value: !Ref AccountTechEmail
        - Key: owner-team-email
          Value: !Ref AccountTeamEmail
        - Key: owner-contact-email
          Value: !Ref AccountOwnerEmail
        - Key: s3_bucket_type
          Value: cloudfront
        - Key: s3_data_retention
          Value: '0'
        - Key: s3_data_classification
          Value: Interna
        - Key: c7n-guardrail-s3-check-security-tags
          Value: https://confluencecorp.ctsp.prod.cloud.ihf/x/KZhXFw
        

  ## Cloud Front Read Policy for Bucket S3
  CloudFrontReadPolicy:
    Type: 'AWS::S3::BucketPolicy'
    Properties:
      Bucket: !Ref s3BUCKET
      PolicyDocument:
        Statement:
        - Sid: 'AllowGetObjCloudFront'
          Action: 's3:GetObject'
          Effect: Allow
          Resource: !Sub 
            - arn:aws:s3:::${NAMEBUCKET}/*
            - { NAMEBUCKET: !Ref s3BUCKET }
          Principal:
            CanonicalUser: !GetAtt CloudFrontOriginAccessIdentity.S3CanonicalUserId
        - Sid: 'AllowSSLRequestOnly'
          Action: 's3:*'
          Effect: 'Deny'
          Resource:
            - !Sub 
              - arn:aws:s3:::${NAMEBUCKET}/*
              - { NAMEBUCKET: !Ref s3BUCKET }
            - !Sub 
              - arn:aws:s3:::${NAMEBUCKET}
              - { NAMEBUCKET: !Ref s3BUCKET }
          Condition:
            Bool:
              'aws:SecureTransport': 'false'
          Principal: '*'

  ## Cloud Front Access Identity
  CloudFrontOriginAccessIdentity:
    Type: AWS::CloudFront::CloudFrontOriginAccessIdentity
    Properties:
      CloudFrontOriginAccessIdentityConfig:
        Comment: !Sub
          - access-identity-${BUCKETDNS}
          - { BUCKETDNS: !GetAtt s3BUCKET.DomainName }

  ## Cloud Front CDN
  CloudFrontDistribution:
    Type: 'AWS::CloudFront::Distribution'
    Properties:
      DistributionConfig:
        Aliases: !Ref CNAMES
        DefaultCacheBehavior:
          AllowedMethods:
          - HEAD
          - GET
          CachedMethods:
          - GET
          - HEAD
          DefaultTTL: 0 # in seconds
          ForwardedValues:
            Cookies:
              Forward: none
            QueryString: false
          MaxTTL: 0 # in seconds
          MinTTL: 0 # in seconds
          TargetOriginId: !Sub 
            - arn:aws:s3:::${NAMEBUCKET}/*
            - { NAMEBUCKET: !Ref s3BUCKET }
          ViewerProtocolPolicy: 'redirect-to-https'
        DefaultRootObject: !Ref DEFAULTROOTOBJECT
        Enabled: true
        ViewerCertificate: 
          AcmCertificateArn: !Ref CERTIFICATE
          MinimumProtocolVersion: "TLSv1.2_2019"
          SslSupportMethod: "sni-only"
        Origins:
        - DomainName: !GetAtt s3BUCKET.DomainName
          Id: !Sub 
            - arn:aws:s3:::${NAMEBUCKET}/*
            - { NAMEBUCKET: !Ref s3BUCKET }
          ConnectionAttempts: 3
          ConnectionTimeout: 10
          OriginPath: !Ref ORIGINPATH
          OriginShield:
            Enabled: true
            OriginShieldRegion: !Ref AWS::Region
          S3OriginConfig:
            OriginAccessIdentity: !Sub 'origin-access-identity/cloudfront/${CloudFrontOriginAccessIdentity}'
        WebACLId: !Ref WAF
        Restrictions:
          GeoRestriction:
            RestrictionType: whitelist
            Locations:
              - AR
              - BR
              - CL
              - CO
              - PY
              - UY
        Logging:
          Bucket: !Sub
            - ${BUCKET}.s3.amazonaws.com
            - { BUCKET: !Ref LOGBUCKET }
          IncludeCookies: false
          Prefix: !Sub
            - cdn-logs-${BUCKET}
            - { BUCKET: !Ref s3BUCKET }
        PriceClass: 'PriceClass_All'
      Tags:
        - Key: CloudFront
          Value: !Ref AWS::StackName
        - Key: Name
          Value: !Sub '${AWS::StackName}-cdn'
        - Key: Region
          Value: !Ref AWS::Region

  iamROLE:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${AWS::StackName}-custom-latam-lambda-role-dynamic-pinning'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: !Sub '${AWS::StackName}-custom-latam-cloudwatch-policy-dynamic-pinning'
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Sid: AllowCreateLogGroup
            Effect: Allow
            Action: logs:CreateLogGroup
            Resource: !Sub 'arn:aws:logs:*:${AWS::AccountId}:*'
          - Sid: AllowCreateLogStream
            Effect: Allow
            Action: logs:CreateLogStream
            Resource: !Sub 'arn:aws:logs:*:${AWS::AccountId}:log-group:/aws/lambda/*:*'
          - Sid: AllowPutLogEvents
            Effect: Allow
            Action: logs:PutLogEvents
            Resource: !Sub 'arn:aws:logs:*:${AWS::AccountId}:log-group:/aws/lambda/*:*'
      - PolicyName: !Sub '${AWS::StackName}-custom-latam-ec2-policy'
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Sid: AllowNetworkInterfaces
            Effect: Allow
            Action:
              - ec2:DescribeNetworkInterfaces
              - ec2:CreateNetworkInterface
              - ec2:DeleteNetworkInterface
            Resource: '*'

  lambdaGETNLBIP:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: 'latam-create-dynamic-pinning'
      Handler: index.handler
      Role: !GetAtt iamROLE.Arn
      Code:
        S3Bucket: !Ref 'BUCKETS3'
        S3Key: !Sub 'deploy/index-${LambdaVersion}.zip'
      Runtime: nodejs14.x
      MemorySize: 1024
      Timeout: 5
      TracingConfig:
        Mode: PassThrough
      VpcConfig:
        SecurityGroupIds:
          - !Ref SECURITYGROUP
        SubnetIds:
          - !Select [0, !Ref SUBNET]
          - !Select [1, !Ref SUBNET]
      Tags:
        - Key: StackName
          Value: !Ref AWS::StackName
        - Key: Region
          Value: !Ref AWS::Region
        - Key: Name
          Value: 'latam-create-dynamic-pinning'
        - Key: Owner
          Value: !Ref AWS::AccountId
        - Key: tech-team-email
          Value: !Ref AccountTechEmail
        - Key: owner-contact-email
          Value: !Ref AccountOwnerEmail

####################
# Outputs section #
####################

Outputs:
  BucketName:
    Description: 'S3 Bucket Name'
    Value: !Ref s3BUCKET
  DistributionId:
    Description: 'CloudFront Distribution ID'
    Value: !Ref CloudFrontDistribution
  Domain:
    Description: 'Cloudfront Domain Name and URL'
    Value: !Sub
      - https://${DNS}
      - { DNS: !GetAtt CloudFrontDistribution.DomainName }
