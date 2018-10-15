import java.text.SimpleDateFormat

node {
    try {
        dir('flask-app') {
            // Get start time of this job - we'll measure the duration
            def measureStartDT = new Date()

            // Image names should be in the format <ERR_URL>/<IMAGE_LABEL>:<IMAGE_VERSION>
            // This will allow them to be pushed and pulled from ECR correctly
            // Image version is appended based on build number
            def imageName = 'xxxxxxxxxxxx.dkr.ecr.us-west-1.amazonaws.com/flask-app' // update the ECR URL here
            def buildNumber = ''
            def imageNameTagged = ''

            // BitBucket repository URL
            def bbRepositoryUrl = 'ssh://bitbucket.org/myteam/myapp.git'            // update the BitBucket URL here

            // Cluster details
            def clusterName = "flask-app-cluster"
            def serviceName = "flask-app-service"
            def taskFamily = "flask-app-task"
            def taskRevision = ''
            def desiredCount = '2'

            // Temp var
            def deploymentMessage = ''
            
            stage('Set up build number') {
                buildNumber = sh(returnStdout: true, script: "printf '%04d' '"+env.BUILD_NUMBER+"'").trim()
                imageNameTagged = imageName+':'+buildNumber
            }
            
            // Code checkout stage....
            stage('Checkout Staging code from BitBucket') {
                // Slack notification:
                slackSend color: 'warning', message: '[flask-app] Check out Staging code from BitBucket'
    
                // Pull the code from BitBucket
                git branch: 'develop', credentialsId: 'bitbucket-login', url: bbRepositoryUrl
            }
        
            // Build and Deploy to Elastic Container Registry stage...
            stage('Build and Push to ECR') {
                // Slack notification:
                slackSend color: 'warning', message: '[flask-app] Build and Push to ECR'

                docker.build(imageNameTagged)

                withCredentials([usernamePassword(credentialsId: 'ecr-flask-app', passwordVariable: 'aws_accesskey', usernameVariable: 'aws_keyid')]) {
                    // Set up AWS CLI
                    sh 'aws configure set aws_access_key_id ${aws_keyid}'
                    sh 'aws configure set aws_secret_access_key ${aws_accesskey}'
                    sh 'aws configure set default.region us-west-1'

                    // Log into ECR
                    sh '$(aws ecr get-login --no-include-email)'

                    // Push image to ECR
                    sh 'docker push '+imageNameTagged
                }
            }

            stage('Set up new task definition and push to service') {
                // Slack notification
                slackSend color: 'warning', message: '[flask-app] Set up new task definition and push to service'

                sh "sed -i -e 's|%BUILD_NUMBER%|${buildNumber}|g' ecs-task.json"
                sh 'aws ecs register-task-definition --cli-input-json file://ecs-task.json'

                // Update the service with the new task definition and desired count
                taskRevision = sh(returnStdout: true, script: "aws ecs describe-task-definition --task-definition ${taskFamily} | egrep 'revision' | tr '/' ' ' | awk '{print \$2}' | sed 's/\"\$//'").trim()
                desiredCount = sh(returnStdout: true, script: "aws ecs describe-services --cluster ${clusterName} --services ${serviceName} | egrep 'desiredCount' | tr '/' ' ' | awk '{print \$2}' | sed 's/,\$//' | head -n 1").trim()
                if (desiredCount == '0') {
                    desiredCount = '1'
                }
                sh "aws ecs update-service --cluster ${clusterName} --service ${serviceName} --task-definition ${taskFamily}:${taskRevision} --desired-count ${desiredCount}"
    
                // Slack notification:
                slackSend color: 'good', message: '[flask-app] Deployment pushed to ECS'
            }

            stage('Wait for ECS to cut over') {
                // Slack notification
                slackSend color: 'warning', message: '[flask-app] Wait for ECS to cut over to new deployment...'

                sh "ls -al"
                withCredentials([usernamePassword(credentialsId: 'ecr-flask-app', passwordVariable: 'aws_accesskey', usernameVariable: 'aws_keyid')]) {
                    deploymentMessage = sh(returnStdout: true, script: "./ecs-wait.sh --cluster '${clusterName}' --service '${serviceName}' --task-definition '${taskFamily}:${taskRevision}' --aws-access-key-id '${aws_keyid}' --aws-secret-access-key '${aws_accesskey}'").trim()
                }

                // Slack notification
                if (deploymentMessage == 'SUCCESS') {
                    slackSend color: 'good', message: '[flask-app] 💃🏻🕺 Deployment pushed to ECS'
                    slackSend color: 'good', message: '[flask-app] Duration: ' + ((new Date().getTime() - measureStartDT.getTime())/1000) + ' seconds.'
                } else {
                    slackSend color: '#f00', message: '[flask-app] Something went wrong with deployment - check ECS!'
                    slackSend color: '#f00', message: '[flask-app] Duration: ' + ((new Date().getTime() - measureStartDT.getTime())/1000) + ' seconds. @channel please check ECS errors.'
                }
            }
        }
    }
    catch (Exception err) {
        // Slack notification:
        slackSend color: '#f00', message: '[API Staging] Deployment failed! ('+err.message+')'
        
        throw err
    }
}