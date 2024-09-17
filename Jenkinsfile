import groovy.json.JsonSlurper

def DEPLOY_BRANCH= 'V1.1.11';
def BASE_VERSION='1.1.11';

def MODULE_NAME = "AR"
def CLUSTER_NAME = ""
def TASK_FAMILY = ""
def SERVICE_NAME ="";
def TASK_DEF_FILE  = "";
def REPO_NAME = "account_receivable_api" ;
def DOCKER_IMAGE_URL = "";


def PASS_KEY="dev.fundflo.ar"

def BUILD_USER_ID ="";
def BUILD_USER_NAME ="";

def APP_NAME='ar-apis';
def AWS_ENV_LOWER='';
def AWS_API_LOWER='';
def DOMAIN_NAME = ".fundflo.ai";

@Library('shared-library')_

pipeline {
  agent any
    
  tools {nodejs "nodejs"}
  options {
    disableConcurrentBuilds()
    timeout(time: 1, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '3'))
  }
  parameters{
    choice(name:'AWS_ENV', choices:['-','UAT'],description:'From which environment do you want to deploy?')
    choice(name:'AWS_API', choices:['DEV','QA','UAT','DEMO'],description:'From which environment do you want to deploy?')
    choice(name:'AWS_REGION', choices:['ap-south-1'],description:'From which region do you want to deploy?')
    choice(name:'SKIP_TEST', choices:['false','true'],description:'Skip Test?')
    choice(name:'BUILD_AND_DEPLOY', choices:['build&deploy','deploy'],description:'Will Build and Deploy or only deploy the latest version?')
    choice(name:'RUN_SQL', choices:['false','true'],description:'Run DML?')
	}

stages {
    stage('GET BUILD INFO & AUTHORIZATION INFO'){
        steps{
          script{
            //  sh 'env'
            //  echo "ENVIRONMENT:  ${env} "    
            AWS_ENV_LOWER = "${params.AWS_ENV}"
            AWS_ENV_LOWER = AWS_ENV_LOWER.toLowerCase()
            AWS_API_LOWER = "${params.AWS_API}"
            AWS_API_LOWER = AWS_API_LOWER.toLowerCase()                
            def cause = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')
            BUILD_USER_ID = "${cause.userId}"
            BUILD_USER_ID = BUILD_USER_ID.replace("[","").replace("]","").trim()
            BUILD_USER_NAME = "${cause.userName}"
            BUILD_USER_NAME = BUILD_USER_NAME.replace("[","").replace("]","").trim()
            
            if(params.AWS_API == 'DEV'){
            }else if(params.AWS_API == 'QA'){
              if(BUILD_USER_ID.equalsIgnoreCase('shankar') || BUILD_USER_ID.equalsIgnoreCase('rattan') || BUILD_USER_ID.equalsIgnoreCase('kumar') || BUILD_USER_ID.equalsIgnoreCase('tushar') || BUILD_USER_ID.equalsIgnoreCase('rajat') || BUILD_USER_ID.equalsIgnoreCase('pramod')){}else{
                currentBuild.result = 'ABORTED'
                echo "ABORTED BUILD...:  ${BUILD_USER_NAME} - ${BUILD_USER_ID} "                      
                throw new RuntimeException("Authorization Issue")
              }
            }else if(params.AWS_API == 'UAT'){
              echo "ENVIRONMENT.....:${params.AWS_ENV} " 
              echo "BUILD_USER_ID...:${BUILD_USER_ID} " 
              if(BUILD_USER_ID.equalsIgnoreCase('shankar') || BUILD_USER_ID.equalsIgnoreCase('rattan') || BUILD_USER_ID.equalsIgnoreCase('rajat') ){}else{
                currentBuild.result = 'ABORTED'
                echo "ABORTED BUILD...:  ${BUILD_USER_NAME} - ${BUILD_USER_ID} "                      
                throw new RuntimeException("Authorization Issue")
              }
            }else if(params.AWS_API == 'DEMO'){
              echo "ENVIRONMENT.....:${params.AWS_ENV} " 
              echo "BUILD_USER_ID...:${BUILD_USER_ID} " 
              if(BUILD_USER_ID.equalsIgnoreCase('shankar') || BUILD_USER_ID.equalsIgnoreCase('rattan') || BUILD_USER_ID.equalsIgnoreCase('kumar') || BUILD_USER_ID.equalsIgnoreCase('rajat') || BUILD_USER_ID.equalsIgnoreCase('tushar') || BUILD_USER_ID.equalsIgnoreCase('pramod')){}else{
                currentBuild.result = 'ABORTED'
                echo "ABORTED BUILD...:  ${BUILD_USER_NAME} - ${BUILD_USER_ID} "                      
                throw new RuntimeException("Authorization Issue")
              }
            }
          }
        }
     }  
      stage('CREATE & PUSH TAG'){
        when {
          expression {env.BRANCH_NAME == DEPLOY_BRANCH }
        }
        steps{
          script{
            VERSION_NO =  gitTag "${BASE_VERSION}";                
            IMAGE_TAG ="${env.BRANCH_NAME}"+'_'+"${VERSION_NO}";
          }
        }
      }
      stage('GET AWS RESOURCE DETAILS OR CREATE'){
        steps{
          script{
              PASS_KEY="${AWS_API_LOWER}.fundflo.ar"
              withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){
              def ecsexists = sh(returnStdout: true, script: "aws ecs describe-clusters --clusters FUND-${params.AWS_API}-$MODULE_NAME-CLUSTER --query 'clusters[0].clusterArn' --output text").trim()
              if (ecsexists!='None'){
                echo "AR ${params.AWS_API} ENVIORNMENT EXISTS"
                sh 'chmod 755 cloud/V1.1.5/aws/aws-data-cli-V5.sh'
                sh 'chmod 755 cloud/aws/aws-data.json'
                sh "./cloud/V1.1.5/aws/aws-data-cli-V5.sh -p ${PASS_KEY}"
                awsDataDetails = readJSON file: 'cloud/aws/aws-data.json'
                echo "awsDataDetails:  ${awsDataDetails} "
                CLUSTER_NAME = "FUND-${params.AWS_API}-$MODULE_NAME-CLUSTER"
                TASK_FAMILY = "FUND-${params.AWS_API}-$MODULE_NAME-TaskDefinition"
                SERVICE_NAME ="FUND-${params.AWS_API}-$MODULE_NAME-SERVICE";
              }else {
                echo "AR ${params.AWS_API} ENVIORNMENT NOT EXISTS"
                sh 'chmod 755 cloud/V1.1.5/aws/aws-data-cli-V5.sh'
                sh 'chmod 755 cloud/aws/aws-data.json'
                sh "./cloud/V1.1.5/aws/aws-data-cli-V5.sh -p ${PASS_KEY}"
                awsDataDetails = readJSON file: 'cloud/aws/aws-data.json'
                echo "awsDataDetails:  ${awsDataDetails} "
                def vpc_id = sh(returnStdout: true, script: "aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=FUND-${params.AWS_ENV}-VPC' --query 'Vpcs[*].{ID:VpcId}' --output text").trim()
                def vpc_cidr = sh(returnStdout: true, script: "aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=FUND-${params.AWS_ENV}-VPC' --query 'Vpcs[0].CidrBlock' --output text").trim()
                def lb_sg_id = sh(returnStdout: true, script: "aws ec2 describe-security-groups --filters 'Name=tag:Name,Values=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-LBSecurityGroup' --query 'SecurityGroups[0].GroupId' --output text").trim()
                def ecs_sg_id = sh(returnStdout: true, script: "aws ec2 describe-security-groups --filters 'Name=tag:Name,Values=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-ContainerSecurityGroup' --query 'SecurityGroups[0].GroupId' --output text").trim()
                if (lb_sg_id!='None'){
                  echo "LOAD BALANCERS SECURITY GROUP ALREADY EXISTS"
                }else {
                  dir ('cloud/V1.1.5/aws/cf_template'){
                    echo "CREATING LOAD BALANCERS SECURITY GROUP ..."
                    sh "aws cloudformation create-stack \
                    --stack-name FUND-${params.AWS_API}-ALB-SG-${MODULE_NAME} \
                    --template-body file://lb_sg.yaml \
                    --parameters ParameterKey=VpcId,ParameterValue=${vpc_id} ParameterKey=SecurityGroupName,ParameterValue=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-LBSecurityGroup\
                    --region ${params.AWS_REGION}"
                    sleep time: 30, unit: 'SECONDS'
                  }
                }
                def lb_sg_id_update = sh(returnStdout: true, script: "aws ec2 describe-security-groups --filters 'Name=tag:Name,Values=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-LBSecurityGroup' --query 'SecurityGroups[0].GroupId' --output text").trim()
                if (ecs_sg_id!='None'){
                  echo "ECS CONTAINER SECURITY GROUP ALREADY EXISTS"
                }else {
                  dir ('cloud/V1.1.5/aws/cf_template'){
                    echo "CREATING ECS CONTAINER SECURITY GROUP ..."
                    sh "aws cloudformation create-stack \
                    --stack-name FUND-${params.AWS_API}-ECS-SG-${MODULE_NAME} \
                    --template-body file://ecs_sg.yaml \
                    --parameters ParameterKey=VpcId,ParameterValue=${vpc_id} ParameterKey=SecurityGroupName,ParameterValue=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-ContainerSecurityGroup ParameterKey=IngressAPIPort,ParameterValue=${awsDataDetails.PORT} ParameterKey=IngressDBPort,ParameterValue=${awsDataDetails.DB_PORT} ParameterKey=CidrRange,ParameterValue=${vpc_cidr} ParameterKey=SourceSecurityGroupId,ParameterValue=${lb_sg_id_update}\
                    --region ${params.AWS_REGION}"
                    sleep time: 30, unit: 'SECONDS'
                  }
                }
                def pb_subnet1a = sh(returnStdout: true, script: "aws ec2 describe-subnets --filters 'Name=tag:Name,Values=FUND-${params.AWS_ENV}-PublicSubnet1a' --query 'Subnets[0].SubnetId' --output text").trim()
                def pb_subnet2b = sh(returnStdout: true, script: "aws ec2 describe-subnets --filters 'Name=tag:Name,Values=FUND-${params.AWS_ENV}-PublicSubnet2b' --query 'Subnets[0].SubnetId' --output text").trim()
                def ecs_subnet1a = sh(returnStdout: true, script: "aws ec2 describe-subnets --filters 'Name=tag:Name,Values=FUND-${params.AWS_ENV}-PrivateSubnet1a' --query 'Subnets[0].SubnetId' --output text").trim()
                def ecs_subnet2b = sh(returnStdout: true, script: "aws ec2 describe-subnets --filters 'Name=tag:Name,Values=FUND-${params.AWS_ENV}-PrivateSubnet2b' --query 'Subnets[0].SubnetId' --output text").trim()
                def lb_sg = sh(returnStdout: true, script: "aws ec2 describe-security-groups --filters 'Name=tag:Name,Values=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-LBSecurityGroup' --query 'SecurityGroups[0].GroupId' --output text").trim()
                def ecs_sg = sh(returnStdout: true, script: "aws ec2 describe-security-groups --filters 'Name=tag:Name,Values=FUND-${params.AWS_API}-ECS-${MODULE_NAME}-ContainerSecurityGroup' --query 'SecurityGroups[0].GroupId' --output text").trim()
                def URI= sh(returnStdout: true, script: "aws secretsmanager get-secret-value --secret-id ${PASS_KEY} | jq -r '.SecretString' | jq --raw-output '.URI'").trim()
                def certificate = sh(returnStdout: true, script: "aws secretsmanager get-secret-value --secret-id ${PASS_KEY} | jq -r '.SecretString' | jq --raw-output '.CERT'").trim()
                dir ('cloud/V1.1.5/aws/cf_template'){
                  echo "CREATING ECS CONTAINER AND LOADBALANCERS..."
                  echo "CREATING ECS CONTAINER SECURITY GROUP ..."
                  sh "aws cloudformation create-stack \
                  --stack-name FUND-${params.AWS_API}-ECS-${MODULE_NAME} \
                  --template-body file://ecs.yaml \
                  --parameters ParameterKey=VPC,ParameterValue=${vpc_id} ParameterKey=SubnetA,ParameterValue=${pb_subnet1a} ParameterKey=SubnetB,ParameterValue=${pb_subnet2b} ParameterKey=ContainerSubnetA,ParameterValue=${ecs_subnet1a} ParameterKey=ContainerSubnetB,ParameterValue=${ecs_subnet2b} ParameterKey=ContainerSecurityGroup,ParameterValue=${ecs_sg} ParameterKey=LoadBalancerSecurityGroup,ParameterValue=${lb_sg} ParameterKey=HostedZoneName,ParameterValue=${AWS_ENV_LOWER}${DOMAIN_NAME} ParameterKey=Subdomain,ParameterValue=${APP_NAME}-${AWS_API} ParameterKey=Image,ParameterValue=${URI} ParameterKey=Certificate,ParameterValue=${certificate} ParameterKey=ServiceName,ParameterValue=FUND-${AWS_API}-${MODULE_NAME} ParameterKey=TaskRole,ParameterValue=FUND-${AWS_ENV}-ECS-TaskRole ParameterKey=AutoScalingRole,ParameterValue=FUND-${AWS_ENV}-ECS-AutoScalingRole ParameterKey=ExecutionRole,ParameterValue=FUND-${AWS_ENV}-ECS-ExecutionRole\
                  --region ${params.AWS_REGION} \
                  --capabilities CAPABILITY_NAMED_IAM"
                  sleep time: 300, unit: 'SECONDS'
                }
                sh 'chmod 755 cloud/V1.1.5/aws/aws-data-cli-V5.sh'
                sh 'chmod 755 cloud/aws/aws-data.json'
                sh "./cloud/V1.1.5/aws/aws-data-cli-V5.sh -p ${PASS_KEY}"
                awsDataDetails = readJSON file: 'cloud/aws/aws-data.json'
                echo "awsDataDetails:  ${awsDataDetails} "
                CLUSTER_NAME = "FUND-${params.AWS_API}-$MODULE_NAME-CLUSTER"
                TASK_FAMILY = "FUND-${params.AWS_API}-$MODULE_NAME-TaskDefinition"
                SERVICE_NAME ="FUND-${params.AWS_API}-$MODULE_NAME-SERVICE";
              }
            }
          }
        }
      }
      stage('Re-runable SQL Script'){
          when {
              expression {env.BRANCH_NAME == DEPLOY_BRANCH }
          }  
        steps{
          script{
            withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){           
              sh 'chmod 755 cloud/run-sql.sh'

              if(params.AWS_ENV == 'DEV' && params.RUN_SQL =='true'){
                sh './cloud/run-sql.sh -u '+"${awsDataDetails.DATABASE_URL}"+' -d '+"${awsDataDetails.DATABASE_NAME}"+' -f V1/db_scripts'
                sh './cloud/run-sql.sh -u '+"${awsDataDetails.DATABASE_URL}"+' -d '+"${awsDataDetails.DATABASE_NAME}"+' -f V1/db_scripts/db_functions'
              }
            }
          }
        }
      }

      stage('INSERT INTO TENANT ENVIRONMENT'){
          when {
              expression {env.BRANCH_NAME == DEPLOY_BRANCH }
          } 
          steps{
            script{
                withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){   
                    DOCKER_IMAGE_URL =  "${awsDataDetails.AWS_ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com";
                    TASK_DEF_FILE = "file://cloud/V1.1.5/aws/task/task-definition-${IMAGE_TAG}.json"
                    TASK_FAMILY_ARN ="arn:aws:ecs:${params.AWS_REGION}:${awsDataDetails.AWS_ACCOUNT_ID}:task-definition/${TASK_FAMILY}"
                }
            }
          }
        }


  
  
      stage('NPM INSTALL'){
        when {
          expression {env.BRANCH_NAME == DEPLOY_BRANCH }
          expression { params.BUILD_AND_DEPLOY != 'deploy' }
        }
        steps{
          script{
            sh "npm cache clean --force"
            sh 'rm -rf package-lock.json && npm install'
            sh 'node --version'

          }
        }
      }
      stage('UPDATE BUILD NUMBER'){
        when {
              expression {env.BRANCH_NAME == DEPLOY_BRANCH }
        }
            steps {
                  script {
                      try{
                        echo "VERSION_NO.... ${VERSION_NO}"
                        updateBuildNumberExtended "${VERSION_NO}";
                      }catch(ex){
                        echo "VERSION_NO EXCEPTION...."+ex
                      }
                }
            }
      }

  /*
      stage('RUN UNIT TEST'){
        when {
          expression { params.AWS_ENV == 'DEV1' && params.BUILD_AND_DEPLOY != 'deploy' && env.BRANCH_NAME == 'dev' && params.SKIP_TEST =='true' }
        }
        steps{
          script{
              try{
                sh "npm run coverage"
              }catch(ex){
                echo ex.message
              }
              publishHTML target: [
                allowMissing: true,
                alwaysLinkToLastBuild: false,
                keepAll: false,
                reportDir: 'coverage/lcov-report',
                reportFiles: 'index.html',
                reportName: 'AR-API-Report'

              ]
            }
          }
        }
  */

      stage('DOCKER LOGIN & PUBLISH'){
        when {
          expression { params.BUILD_AND_DEPLOY == 'build&deploy' && env.BRANCH_NAME == DEPLOY_BRANCH }
        }
        steps{
          script {
            withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){ 
              ECR_CREDENTIALS = sh(script: 'aws ecr get-login --no-include-email', returnStdout: true)
              sh(script: "${ECR_CREDENTIALS}", returnStdout: false)
                        
              // Building & Tagging Docker Image
              sh "docker system prune -f"
              try{
                sh "docker images --no-trunc --format '{{.ID}}' | xargs docker rmi -f"
              }catch(err){
                  echo "ERROR in removing images"+err
              }
              sh "docker build -t ${DOCKER_IMAGE_URL}/${REPO_NAME} . --no-cache"
              sh "docker tag ${DOCKER_IMAGE_URL}/${REPO_NAME} ${DOCKER_IMAGE_URL}/${REPO_NAME}:${IMAGE_TAG}"
            // Pushing to ECR
              sh "docker push ${DOCKER_IMAGE_URL}/${REPO_NAME}:${IMAGE_TAG} "
            // Logout Docker
              sh(script: "docker logout ${DOCKER_IMAGE_URL}", returnStdout: true)
            }
          }
        }
      }

      stage('Task Update') {
          when {
              expression {env.BRANCH_NAME == DEPLOY_BRANCH }
          } 
        steps{
          script {
            withAWS(region:"${params.AWS_REGION}",credentials:"${params.AWS_ENV}"){ 
              try{
                    sh  "sed -e  's#%DB_HOST%#${awsDataDetails.SECRET_ARN}:DB_HOST::#g; \
                    s#%DB_PORT%#${awsDataDetails.SECRET_ARN}:DB_PORT::#g; \
                    s#%DB_USER%#${awsDataDetails.SECRET_ARN}:DB_USER::#g; \
                    s#%DB_PASS%#${awsDataDetails.SECRET_ARN}:DB_PASS::#g; \
                    s#%DB_NAME%#${awsDataDetails.SECRET_ARN}:DB_NAME::#g; \
                    s#%SMS_URL%#${awsDataDetails.SECRET_ARN}:SMS_URL::#g; \
                    s#%SMS_USER%#${awsDataDetails.SECRET_ARN}:SMS_USER::#g; \
                    s#%SMS_PASS%#${awsDataDetails.SECRET_ARN}:SMS_PASS::#g; \
                    s#%OTP_URL%#${awsDataDetails.SECRET_ARN}:OTP_URL::#g; \
                    s#%OTP_PASS%#${awsDataDetails.SECRET_ARN}:OTP_PASS::#g; \
                    s#%OTP_EXP%#${awsDataDetails.SECRET_ARN}:OTP_EXP::#g; \
                    s#%PORT%#${awsDataDetails.SECRET_ARN}:PORT::#g; \
                    s#%NODE_ENV%#${awsDataDetails.SECRET_ARN}:NODE_ENV::#g; \
                    s#%ROLE_ARN%#${awsDataDetails.SECRET_ARN}:ROLE_ARN::#g; \
                    s#%SQS_URL%#${awsDataDetails.SECRET_ARN}:SQS_URL::#g; \
                    s#%SQS_REGION%#${awsDataDetails.SECRET_ARN}:SQS_REGION::#g; \
                    s#%SEND_HEALTHCHECK%#${awsDataDetails.SECRET_ARN}:SEND_HEALTHCHECK::#g; \
                    s#%LSQ_SYNC_CRON%#${awsDataDetails.SECRET_ARN}:LSQ_SYNC_CRON::#g; \
                    s#%NOTIFICATION_URL%#${awsDataDetails.SECRET_ARN}:NOTIFICATION_URL::#g; \
                    s#%NOTIFICATION_API_KEY%#${awsDataDetails.SECRET_ARN}:NOTIFICATION_API_KEY::#g; \
                    s#%FILES_BUCKET_NAME%#${awsDataDetails.SECRET_ARN}:FILES_BUCKET_NAME::#g; \
                    s/%MODULE_NAME%/${MODULE_NAME}/g; \
                    s/%CLOUD_ENV%/${params.AWS_ENV}/g; \
                    s/%API_ENV%/${params.AWS_API}/g; \
                    s/%AWS_ACCOUNT_ID%/${awsDataDetails.AWS_ACCOUNT_ID}/g; \
                    s/%DOCKER_IMAGE_URL%/${DOCKER_IMAGE_URL}/g; \
                    s/%REPO_NAME%/${REPO_NAME}/g; \
                    s/%BUILD_TAG%/${IMAGE_TAG}/g; \
                    s/%CLOUD_REGION%/${params.AWS_REGION}/g;' cloud/V1.1.5/aws/task/task-definition.json > cloud/V1.1.5/aws/task/task-definition-${IMAGE_TAG}.json"                    

                      def currTaskDef = sh (
                        returnStdout: true,
                        script:  "aws ecs describe-task-definition  --task-definition ${TASK_FAMILY_ARN}  | egrep 'revision' | tr ',' ' ' | awk '{print \$2}'").trim()

                      def currentTaskList = "aws ecs list-tasks --cluster ${CLUSTER_NAME} | jq -r '.taskArns[]'"
                      if(currTaskDef) {
                        echo "Current Task Def ${currTaskDef}"
                        sh  "aws ecs update-service  --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${TASK_FAMILY_ARN}:${currTaskDef} --desired-count 1"
                      }
                      // // Register the new [TaskDefinition]
                      sh  "aws ecs register-task-definition  --family ${TASK_FAMILY} --cli-input-json ${TASK_DEF_FILE}"
                      // Get the last registered [TaskDefinition#revision]
                      def taskRevision = sh (
                        returnStdout: true,
                        script:  "aws ecs describe-task-definition  --task-definition ${TASK_FAMILY_ARN}  | egrep 'revision' | tr ',' ' ' | awk '{print \$2}'").trim()
                      echo "taskRevision:::${taskRevision}"
                      // ECS update service to use the newly registered [TaskDefinition#revision]
                      sh  "aws ecs update-service  --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${TASK_FAMILY_ARN}:${taskRevision} --desired-count 1 "
                    if (params.AWS_ENV == 'DEV') {
                      def deregisterTask = sh (
                        returnStdout: true,
                        script:  "aws ecs deregister-task-definition --task-definition ${TASK_FAMILY}:${currTaskDef}")

                      // def deleteTask = sh (
                      //   returnStdout: true,
                      //   script:  "aws ecs delete-task-definitions --task-definition ${TASK_FAMILY}:${currTaskDef}")
                    }

              }catch(err){
                echo "Deployment in TASK DEF FAILED"+err
                throw err;
              }
            }
          }
        }
      }
      stage('CLEAN UP'){
        steps{
                sh "rm -rf node_modules"
                sh "rm -rf package-lock.json"
        }
      }
    }

    post {
      // always {
      //         publishHTML target: [
      //           allowMissing: true,
      //           alwaysLinkToLastBuild: false,
      //           keepAll: false,
      //           reportDir: 'coverage/lcov-report',
      //           reportFiles: 'index.html',
      //           reportName: 'Test-Report'
      //         ]
      //   }
        success {
          slackSend botUser: true, 
            channel: '#jenkinssuccess', 
            color: '#095e0b', 
            message: "SUCCESSFULLY BUILD BY : ${BUILD_USER_NAME}(${BUILD_USER_ID}) WITH JOB  ${env.JOB_NAME}(BUILD-NO: ${env.BUILD_NUMBER}) (<${env.BUILD_URL}|JENKINS-LINK>) -- (<https://${APP_NAME}-${AWS_API_LOWER}.${AWS_ENV_LOWER}${DOMAIN_NAME}|API-LINK>)", 
            tokenCredentialId: 'slack-token'
        }
        failure {        
          slackSend botUser: true, 
            channel: '#jenkinsfailure', 
            color: '#f50a0a', 
            message: "FAILED BUILD..... ${BUILD_USER_NAME}|${BUILD_USER_ID} - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", 
            tokenCredentialId: 'slack-token'
        }
        aborted {
          slackSend botUser: true, 
          channel: '#jenkinsfailure', 
          color: '#939693', 
            message: "ABORTED BUILD..... ${BUILD_USER_NAME}|${BUILD_USER_ID} - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", 
          tokenCredentialId: 'slack-token'
          }
    }


  }
