pipeline {
    agent any

    environment {
        DOCKER_ID = credentials('DOCKER_ID')
        DOCKER_PASSWORD = credentials('DOCKER_PASSWORD')
        AWS_DEFAULT_REGION="us-east-1"
        THE_BUTLER_SAYS_SO=credentials('cmb-aws-cred')
        BUILD_ID = "${env.BUILD_ID}"
    }

    stages {
        stage('Init') {
            steps {
                echo 'Initializing..'
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
                echo "Current branch: ${env.BRANCH_NAME}"
                sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_ID --password-stdin'
            }
        }
        stage('AWS') {           
            steps {
                echo 'AWS command..'
                sh '''
                  aws --version
                  aws ec2 describe-instances
                '''
            }
        }
        stage ('push artifact') {
            steps {
                    sh 'zip -r index-${BUILD_ID}.zip src'
                    sh 'aws s3 cp $WORKSPACE/index-${BUILD_ID}.zip s3://create-lambda-from-zip-file/'
                    // archiveArtifacts artifacts: 'index-${BUILD_ID}.zip', fingerprint: true
            }
        }      
//        stage('new pull artifact new') {
//            steps {                
//                    copyArtifacts filter: 'test13.zip', fingerprintArtifacts: true, projectName: env.JOB_NAME, selector: specific(env.BUILD_NUMBER)
//                //  unzip zipFile: 'test11.zip', dir: './archive_new'
//                    sh 'unzip test13.zip -d ./archive_new'
//                    sh 'cat archive_new/test13.txt'                
//            }
//        }        
        stage('Deploy') {
            steps {
                echo 'Removing unused docker containers and images..'
                // sh "aws cloudformation create-stack --stack-name s3bucket --template-body file://simplests3cft.json --region 'us-east-1'"
                // sh "aws cloudformation create-stack --stack-name lambdafroms3bucket --template-body file://lambda.yaml --region 'us-east-1'"
                // sh "aws cloudformation create-stack --stack-name lambdafroms3bucket --region us-east-1 --template-body file://lambda.yml --parameters file://param.json --capabilities "CAPABILITY_IAM" "CAPABILITY_NAMED_IAM""
                sh 'aws cloudformation create-stack --stack-name LambdaFromBucket --region us-east-1 --template-body file://lambda.yaml --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM '
            }
        }
    }
}
