import groovy.json.JsonSlurper

def S3_BUCKET_NAME          = "opsflex-cicd-mgmt"
def S3_PATH                 = "frontend"
def S3_FILE_NAME            = "frontend.zip"

def TARGET_DOMAIN_NAME      = ""
def TARGET_GROUP_PREFIX     = ""
def TARGET_RULE_ARN         = ""
def ALB_LISTENER_ARN        = ""

def DEPLOY_GROUP_NAME       = ""
def DEPLOYMENT_ID           = ""
def ASG_DESIRED             = 0
def ASG_MIN                 = 1
def CURR_ASG_NAME           = ""
def NEXT_ASG_NAME           = ""
def NEXT_TG_ARN             = ""
def ALB_ARN                 = ""
def TG_RULE_ARN             = ""

def NPM_MODE                = ""

def CODE_DEPLOY_NAME        = ""

@NonCPS
def toJson(String text) {
    def parser = new JsonSlurper()
    return parser.parseText( text )
}

def initEnvironment(String text) {
    echo "initEnvironment param : ${text}"
    if (text == 'develop') {
        env.NPM_MODE = 'build-dev'
        env.TARGET_DOMAIN_NAME    = "dev.ui.mingming.shop"
        env.TARGET_GROUP_PREFIX   = "demo-apne2-dev-ui"
        env.ALB_LISTENER_ARN    = "arn:aws:elasticloadbalancing:ap-northeast-2:144149479695:listener/app/comp-apne2-prod-mgmt-alb/d76ec25af38db29c/d15a5636f3b71341"
        env.CODE_DEPLOY_NAME      = "demo-apne2-ui-codedeploy"
    } else if (text == 'release') {
        env.NPM_MODE = 'build-rel'
        env.TARGET_DOMAIN_NAME    = "dev.ui.mingming.shop"
        env.TARGET_GROUP_PREFIX   = "demo-apne2-dev-ui"
        env.ALB_LISTENER_ARN      = "arn:aws:elasticloadbalancing:ap-northeast-2:144149479695:listener/app/comp-apne2-prod-mgmt-alb/d76ec25af38db29c/d15a5636f3b71341"
        env.CODE_DEPLOY_NAME      = "demo-apne2-ui-codedeploy"
    } else if (text == 'master') {
        env.NPM_MODE = 'build-prod'
        env.TARGET_DOMAIN_NAME    = "dev.ui.mingming.shop"
        env.TARGET_GROUP_PREFIX   = "demo-apne2-dev-ui"
        env.ALB_LISTENER_ARN      = "arn:aws:elasticloadbalancing:ap-northeast-2:144149479695:listener/app/comp-apne2-prod-mgmt-alb/d76ec25af38db29c/d15a5636f3b71341"
        env.CODE_DEPLOY_NAME      = "demo-apne2-ui-codedeploy"
    } else {
        env.NPM_MODE = ""
        error "env parameter error!!!!"
    }
}

def showVariables() {
  echo """
>   NPM_MODE:               ${env.NPM_MODE}
    TARGET_DOMAIN_NAME:     ${env.TARGET_DOMAIN_NAME}
    TARGET_GROUP_PREFIX:    ${env.TARGET_GROUP_PREFIX}
    TARGET_RULE_ARN:        ${env.TARGET_RULE_ARN}
    CURR_ASG_NAME:          ${env.CURR_ASG_NAME}
    NEXT_ASG_NAME:          ${env.NEXT_ASG_NAME}
    DEPLOY_GROUP_NAME:      ${env.DEPLOY_GROUP_NAME}
    ALB_ARN:                ${env.ALB_ARN}
    NEXT_TG_ARN:            ${env.NEXT_TG_ARN}
    NEXT_TARGET_GROUP:      ${env.NEXT_TARGET_GROUP}
    """
}

def initVariables(def tgList) {
    tgList.each { tg ->
        String lbARN  = tg.LoadBalancerArns[0]
        String tgName = tg.TargetGroupName
        String tgARN  = tg.TargetGroupArn
        // loadbalncer targetList
        if(lbARN != null && lbARN.startsWith("arn:aws")) {
            env.ALB_ARN = lbARN
            if(tgName.startsWith("${env.TARGET_GROUP_PREFIX}" + "-a")) {
                echo ">> Init change from group a to group b"
                env.DEPLOY_GROUP_NAME = "demo-ui-group-b"
                env.CURR_ASG_NAME     = env.TARGET_GROUP_PREFIX + "-a-asg"
                env.NEXT_ASG_NAME     = env.TARGET_GROUP_PREFIX + "-b-asg"
            } else {
                echo ">> Init change from group b to group a"
                env.DEPLOY_GROUP_NAME = "demo-ui-group-a"
                env.CURR_ASG_NAME     = env.TARGET_GROUP_PREFIX + "-b-asg"
                env.NEXT_ASG_NAME     = env.TARGET_GROUP_PREFIX + "-a-asg"
            }
        } else {
            env.NEXT_TG_ARN       = tgARN
            env.NEXT_TARGET_GROUP = tgName
        }
    }
}

def getBuildPath() {
    return "${JOB_NAME}".replaceAll('/','_')
}

def discoveryTargetRuleArn(def listenerARN, def tgPrefix) {
  script {
    return sh(
      script: """aws elbv2 describe-rules --listener-arn ${listenerARN} \
                   --query 'Rules[].{RuleArn: RuleArn, Actions: Actions[?contains(TargetGroupArn,`${tgPrefix}`)==`true`]}' \
                   --region ap-northeast-2 \
                   --output text | grep -B1 "ACTIONS"  | grep -v  "ACTIONS"   """, 
      returnStdout: true).trim()
    }
}

pipeline {
    agent any
    stages {
        stage('Pre-Process') {
            steps {
                script {
                    
                    echo "----- [Pre-Process] init Environment -----"
                    initEnvironment("${GIT_BRANCH}")
                    
                    echo "----- [Pre-Process] target arn setting ----"
                    def target_rule_arn = discoveryTargetRuleArn(env.ALB_LISTENER_ARN, env.TARGET_GROUP_PREFIX)
                    env.TARGET_RULE_ARN = target_rule_arn
                    
                    echo "----- [Pre-Process] showVariables -----"
                    showVariables()
                    
                    echo "----- [Pre-Process] Discovery Active Target Group -----"
                    sh"""
                    aws elbv2 describe-target-groups \
                    --query 'TargetGroups[?starts_with(TargetGroupName,`${env.TARGET_GROUP_PREFIX}`)==`true`]' \
                    --region ap-northeast-2 --output json > TARGET_GROUP_LIST.json
                    cat ./TARGET_GROUP_LIST.json
                    """
        
                    def textValue = readFile("TARGET_GROUP_LIST.json")
                    def tgList = toJson(textValue)
                    
                    echo "----- [Pre-Process] Initialize Variables -----"
                    initVariables(tgList)
                    
                    echo "----- [Pre-Process] showVariables -----"
                    showVariables()
                }
            }
        }
        stage('Check-current autoscaling') {
            steps {
                script {
                    echo "----- [Check-current autoscaling] Autoscaling instances setting -----"
                    sh """
                    aws autoscaling describe-auto-scaling-groups --query 'AutoScalingGroups[?starts_with(AutoScalingGroupName,`${env.CURR_ASG_NAME}`)==`true`].Instances[?LifecycleState==InService]' \
                    --region ap-northeast-2 \
                    --output text |grep InService | wc -l > ASG_DESIRED_CNT.json
                    """
                    def desiredAsg = readFile("ASG_DESIRED_CNT.json").toInteger()
                    if ("${desiredAsg}" > 0) {
                        env.ASG_DESIRED = "${desiredAsg}"
                    } else {
                        env.ASG_DESIRED = 1
                    }
                    echo "autoscaling instances size : ${env.ASG_DESIRED}"
                }
            }
        }
        
        stage('Git clone') {
            steps {
                script {
                    def scmUrl = scm.getUserRemoteConfigs()[0].getUrl()
                    git(url: "${scmUrl}", branch: "${GIT_BRANCH}", changelog: true)
                }
            }
        }
        
        stage ('Npm build') {
            steps {
                sh "echo ----- [npm build] -----"
                // directory check
                sh '''
                pwd
                npm install
                echo npm install success
                '''
                sh "npm run ${env.NPM_MODE}"
                // codedeploy setting
                sh "cp -rf /var/lib/jenkins/workspace/${getBuildPath()}/scripts /var/lib/jenkins/workspace/${getBuildPath()}/dist"
                sh "cp -rf /var/lib/jenkins/workspace/${getBuildPath()}/appspec.yml /var/lib/jenkins/workspace/${getBuildPath()}/dist" 
            }
        }
        
        stage ('S3 upload') {
            steps {
                sh "echo ----- [S3 Upload] -----"
                dir ("/var/lib/jenkins/workspace/${getBuildPath()}/dist") {
                    sh "jar cvf ${S3_FILE_NAME} *"
                    sh "ls"
                    withAWS(region:'ap-northeast-2') {
                        s3Upload(file:"${S3_FILE_NAME}", bucket:"${S3_BUCKET_NAME}", path:"${S3_PATH}/${getBuildPath()}/${BUILD_NUMBER}/${S3_FILE_NAME}")
                    }      
                }
            }
        }
        
        stage('Preparing Auto-Scale Stage') {
            steps {
                script {
                    echo "----- [Auto-Scale] Preparing Next Auto-Scaling-Group: ${env.NEXT_ASG_NAME} -----"
        
                    sh"""
                    aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${env.NEXT_ASG_NAME} \
                    --desired-capacity ${env.ASG_DESIRED} \
                    --min-size ${ASG_MIN} \
                    --region ap-northeast-2
                    """
        
                    echo "----- [Auto-Scale] Waiting boot-up ec2 instances: 90 secs. -----"
                    sh "sleep 90"
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    sh "echo ----- [Codedeploy] -----"
                    dir ("/var/lib/jenkins/workspace/${getBuildPath()}/dist") {
                        sh """
                        aws deploy create-deployment \
                        --application-name "${env.CODE_DEPLOY_NAME}" \
                        --s3-location bucket="${S3_BUCKET_NAME}",key=${S3_PATH}/${getBuildPath()}/${BUILD_NUMBER}/${S3_FILE_NAME},bundleType=zip \
                        --deployment-group-name "${env.DEPLOY_GROUP_NAME}" \
                        --description "create frontend" \
                        --region ap-northeast-2 \
                        --output json > DEPLOYMENT_ID.json
                        """
                        
                        def textValue = readFile("DEPLOYMENT_ID.json")
                        def jsonDI =toJson(textValue)
                        env.DEPLOYMENT_ID = "${jsonDI.deploymentId}"
                    }
                }
            }
        }
        
        stage('Health-Check') {
            steps {
                script {
                    echo "----- [Health-Check] DEPLOYMENT_ID ${env.DEPLOYMENT_ID} -----"
                    echo "----- [Health-Check] Waiting codedeploy processing -----"
                    timeout(time: 3, unit: 'MINUTES'){                                         
                      awaitDeploymentCompletion("${env.DEPLOYMENT_ID}")
                    }
                }
            }
        }
    
        stage('Change LB-Routing') {
            steps {
                script {
                    echo "----- [LB] Change load-balancer routing path -----"
                    sh"""
                    aws elbv2 modify-rule --rule-arn ${env.TARGET_RULE_ARN} \
                    --conditions Field=host-header,Values=${env.TARGET_DOMAIN_NAME} \
                    --actions Type=forward,TargetGroupArn=${env.NEXT_TG_ARN} \
                    --region ap-northeast-2 --output json > CHANGED_LB_TARGET_GROUP.json
                    cat ./CHANGED_LB_TARGET_GROUP.json
                    """
                }   
            }
        }
    
        stage('Stopping Blue Stage') {
            steps {
                script{
                    echo "----- [Stopping Blue] Stopping Blue Stage. Auto-Acaling-Group: ${env.CURR_ASG_NAME} -----"
                    sh"sleep 30"
                    sh"""
                    aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${env.CURR_ASG_NAME}  \
                    --desired-capacity 0 --min-size 0 \
                    --region ap-northeast-2
                    """
                }
            }
        }
    }
}
