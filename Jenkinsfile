def stageName = ''

def stepError = ''

def taskList = ''

def serviceName = 'E2E-test-automation'

def serviceGroup = 'platform'



pipeline {

    agent none

    stages {

        stage('PullRequest') {

            agent { label 'built-in' }

            when {

                changeRequest target: 'master'

            }

            stages {

                stage('Test Build For PR') {

                    steps {

                        script { 

                            stageName = 'PullRequest'

                        }

                        buildImage("test", serviceName)

                    }

                }

            }

        }

        stage ('Development') {

            agent { label 'built-in' }

            when{

                branch 'master'

            }

            stages {

                stage('Build') {

                    steps {

                        script { 

                            stepError = 'Build the docker image has been failed.'

                            stageName = 'Development'

                        }

                        notifyBuild("STARTED", stageName, "Start Building The Image")

                        buildImage("dev", serviceName)

                    }

                }

                stage('Deploy') {

                    steps {

                        script { 

                            stepError = 'Deploying the latest image to ECS has been failed.'

                            stageName = 'Development'

                        }

                        deployFeatures("dev", serviceName, serviceGroup)

                    }

                }

            }

            post { 

                success {

                    script {

                        taskList = jiraTaskProgress(stageName)

                    }

                    notifyBuild("SUCCEED", stageName, "Features Deployed", stepError, taskList)

                }

            }

        }

        stage ('Staging') {

            agent { label 'built-in' }

            when{

                tag "*"

            }

            stages {

                stage('Build') {

                    steps {

                        script { 

                            stepError = 'Build the docker image has been failed.'

                            stageName = 'Staging'

                        }

                        notifyBuild("STARTED", stageName, "Start Building The Image")

                        buildImage("stage", serviceName)

                    }

                }

                stage('Deploy') {

                    steps {

                        script { 

                            stepError = 'Deploying the latest image to ECS has been failed.'

                            stageName = 'Staging'

                        }

                        deployFeatures("stage", serviceName, serviceGroup)

                    }

                }

            }

            post { 

                success {

                    script {

                        taskList = jiraTaskProgress(stageName)

                    }

                    notifyBuild("SUCCEED", stageName, "Features Deployed", stepError, taskList)

                }

            }

        }

/*        

        stage('Deploy Approval'){

            when{

                tag "*"

            }

            steps {

                #script { stageName = 'Production' }

                notifyBuild("APPROVE", stageName, "Approve To Deploy Changes")

                input ("Do you want to deploy changes to production?")

            }

        }

*/        

        stage('Deploy Production') {

            agent { label 'built-in' }

            when{

                tag "*"

            }

            environment {

                AWS_PROFILE = "prod"

            }

            steps {

                script {

                    stepError = 'Deploying the latest image to ECS has been failed.'

                    stageName = 'Production'

                }

                deployFeatures("prod", serviceName, serviceGroup)

            }

            post { 

                success {

                    script {

                        taskList = jiraTaskProgress(stageName)

                    }

                    notifyBuild("SUCCEED", stageName, "Features Deployed", stepError, taskList)

                }

            }

        }

    }

    post { 

        failure { 

            notifyBuild("FAILED", stageName, "Features Deployed", stepError)

        }

    }

}



def notifyBuild(String buildStatus = 'STARTED', String stage = 'Development', String action = 'Deploy', String ciError = 'Error', String taskList = '') {

    def colorCode = ''

    def channelName = ''



    // Skip sending message for pull requests

    if (stage == "PullRequest") {

        return true;

    }



    // Set message color in slack

    switch(buildStatus) {

    case "STARTED":

        colorCode = "#2761FF"

        break

    case "APPROVE":

        colorCode = "#FFAD27"

        break

    case "FAILED":

        colorCode = "#FF0000"

        break

    default:

        colorCode = "#13AD00"

        break

    }



    // Set Channel name according to the stage name

    switch(stage) {

    case "Development":

        channelName = "#deployment-dev"

        break

    case "Staging":

        channelName = "#deployment-staging"

        break

    case "Production":

        channelName = "#deployment-prod"

        break

    default:

        channelName = "#deployment-dev"

        break

    }



    // Create a message with all inputs

    def message = "Job: *${env.JOB_NAME}*\nEnvironment: *${stage}*\nStatus: *${buildStatus}*\nAction: *${action}*"

    

    // Add error message only if status is failed 

    if(buildStatus == 'FAILED') {

        message = message + "\nError description: *${ciError}*"

    }

    message = message + "\n Build Report: <${env.RUN_DISPLAY_URL}|Jump To Console>\n${taskList}"



    slackSend(color: colorCode, message: message, channel: channelName)

}



def jiraTaskProgress(String stage = 'Development') {

    def transitionNumber = ''

    def getTaskNumberCommand = ''

    def slackMessage = '*** Jira Tasks *** \n'



    // Set transition number and Command according to the stage name

    if (stage == "Development") {

        transitionNumber = "101"

        getTaskNumberCommand = "git log --merges -n1 --format='%s' | grep -oP 'PX-[0-9]*' || echo ''"

    } else {

        transitionNumber = "211"

        def tagName = sh(script: "git tag --sort=-\"v:refname\" | head -n2 | grep -v ${env.TAG_NAME}", returnStdout: true).toString().replace('\n', '')

        getTaskNumberCommand = "git log --merges ${tagName}..HEAD --format='%s' | grep -oP 'PX-[0-9]*' | uniq || echo ''"

    }

    

    // Get the task number from merged pull request title

    def taskNumber = sh(script: "${getTaskNumberCommand}", returnStdout: true).toString()

    if (taskNumber != '') {

        taskList = taskNumber.split('\n')

        for (task in taskList) {

            if (stage != 'Staging') {

                jiraChangeTaskStatus(task, transitionNumber)

            }

            def title = jiraTaskTitle(task).toString().replace('\n', '')

            def link = "https://finlex.atlassian.net/browse/${task}"

            slackMessage += "*${task}:* <${link}|${title}> \n"

        }

        return slackMessage

    } else {

        return "Get Task Failed: *There is no valid task for this merged PR.*"

    }

}



def jiraTaskTitle(String taskNumber) {

    def taskTitle = ''

    // Get task title from the Jira API endpoint

    withCredentials([string(credentialsId: 'JiraAPIToken', variable: 'password')]) {

        taskTitle = sh(script: "curl -XGET https://finlex.atlassian.net/rest/api/3/issue/${taskNumber} -u it@finlex.de:${password} | jq -r '.fields.summary'", returnStdout: true).toString()

    }

    return taskTitle

}



def jiraChangeTaskStatus(String taskNumber, String transitionNumber) {

    // Change task status with Jira API endpoint according to the stage

    withCredentials([string(credentialsId: 'JiraAPIToken', variable: 'password')]) {

        sh "curl -XPOST https://finlex.atlassian.net/rest/api/3/issue/${taskNumber}/transitions -u 'it@finlex.de:${password}' --data '{\"transition\":{\"id\":\"${transitionNumber}\"}}'"

    }

}



def buildImage(String stage = 'dev', String serviceName) {

    def dockerTagName = 'dev'

    // Decided which branch should be build according to the environment

    if (stage == 'dev' || stage == 'test') {

        dockerTagName = stage

    } else {

        dockerTagName = env.TAG_NAME

    }

    sh '$(aws ecr get-login --no-include-email --registry-ids 474975247937 --region eu-central-1)'

    sh "docker build -t 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${dockerTagName} ."

    if (stage != 'test') {

        sh "docker push 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${dockerTagName}"

    }

}



def deployFeatures(String stage = 'dev', String serviceName, String serviceGroup) {

    def dockerTagName = 'dev'

    def zone = ''

    def cluster = ''



    // According to the environment set different varibale

    if (stage == 'dev') {

        dockerTagName = 'dev'

        zone = 'eu-west-1'

        cluster = 'dev0'

    } else if (stage == 'stage') {

        dockerTagName = env.TAG_NAME

        zone = 'eu-central-1'

        cluster = 'staging1'

    } else {

        dockerTagName = env.TAG_NAME

        zone = 'eu-central-1'

        cluster = 'prod'

    }



    sh '$(aws ecr get-login --no-include-email --registry-ids 474975247937 --region eu-central-1)'

    // if the environment is not development, we need to tagged version to the stage name

    if (stage != 'dev') {

        sh "docker pull 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${env.TAG_NAME}"

        sh "docker tag 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${env.TAG_NAME} 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${stage}"

        sh "docker push 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${stage}"

    }

    sh "docker run 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${env.TAG_NAME}

    

}
Added file
Jenkinsfile



@@ -0,0 +1,286 @@

def stageName = ''

def stepError = ''

def taskList = ''

def serviceName = 'E2E-test-automation'

def serviceGroup = 'platform'



pipeline {

    agent none

    stages {

        stage('PullRequest') {

            agent { label 'built-in' }

            when {

                changeRequest target: 'master'

            }

            stages {

                stage('Test Build For PR') {

                    steps {

                        script { 

                            stageName = 'PullRequest'

                        }

                        buildImage("test", serviceName)

                    }

                }

            }

        }

        stage ('Development') {

            agent { label 'built-in' }

            when{

                branch 'master'

            }

            stages {

                stage('Build') {

                    steps {

                        script { 

                            stepError = 'Build the docker image has been failed.'

                            stageName = 'Development'

                        }

                        notifyBuild("STARTED", stageName, "Start Building The Image")

                        buildImage("dev", serviceName)

                    }

                }

                stage('Deploy') {

                    steps {

                        script { 

                            stepError = 'Deploying the latest image to ECS has been failed.'

                            stageName = 'Development'

                        }

                        deployFeatures("dev", serviceName, serviceGroup)

                    }

                }

            }

            post { 

                success {

                    script {

                        taskList = jiraTaskProgress(stageName)

                    }

                    notifyBuild("SUCCEED", stageName, "Features Deployed", stepError, taskList)

                }

            }

        }

        stage ('Staging') {

            agent { label 'built-in' }

            when{

                tag "*"

            }

            stages {

                stage('Build') {

                    steps {

                        script { 

                            stepError = 'Build the docker image has been failed.'

                            stageName = 'Staging'

                        }

                        notifyBuild("STARTED", stageName, "Start Building The Image")

                        buildImage("stage", serviceName)

                    }

                }

                stage('Deploy') {

                    steps {

                        script { 

                            stepError = 'Deploying the latest image to ECS has been failed.'

                            stageName = 'Staging'

                        }

                        deployFeatures("stage", serviceName, serviceGroup)

                    }

                }

            }

            post { 

                success {

                    script {

                        taskList = jiraTaskProgress(stageName)

                    }

                    notifyBuild("SUCCEED", stageName, "Features Deployed", stepError, taskList)

                }

            }

        }

/*        

        stage('Deploy Approval'){

            when{

                tag "*"

            }

            steps {

                #script { stageName = 'Production' }

                notifyBuild("APPROVE", stageName, "Approve To Deploy Changes")

                input ("Do you want to deploy changes to production?")

            }

        }

*/        

        stage('Deploy Production') {

            agent { label 'built-in' }

            when{

                tag "*"

            }

            environment {

                AWS_PROFILE = "prod"

            }

            steps {

                script {

                    stepError = 'Deploying the latest image to ECS has been failed.'

                    stageName = 'Production'

                }

                deployFeatures("prod", serviceName, serviceGroup)

            }

            post { 

                success {

                    script {

                        taskList = jiraTaskProgress(stageName)

                    }

                    notifyBuild("SUCCEED", stageName, "Features Deployed", stepError, taskList)

                }

            }

        }

    }

    post { 

        failure { 

            notifyBuild("FAILED", stageName, "Features Deployed", stepError)

        }

    }

}



def notifyBuild(String buildStatus = 'STARTED', String stage = 'Development', String action = 'Deploy', String ciError = 'Error', String taskList = '') {

    def colorCode = ''

    def channelName = ''



    // Skip sending message for pull requests

    if (stage == "PullRequest") {

        return true;

    }



    // Set message color in slack

    switch(buildStatus) {

    case "STARTED":

        colorCode = "#2761FF"

        break

    case "APPROVE":

        colorCode = "#FFAD27"

        break

    case "FAILED":

        colorCode = "#FF0000"

        break

    default:

        colorCode = "#13AD00"

        break

    }



    // Set Channel name according to the stage name

    switch(stage) {

    case "Development":

        channelName = "#deployment-dev"

        break

    case "Staging":

        channelName = "#deployment-staging"

        break

    case "Production":

        channelName = "#deployment-prod"

        break

    default:

        channelName = "#deployment-dev"

        break

    }



    // Create a message with all inputs

    def message = "Job: *${env.JOB_NAME}*\nEnvironment: *${stage}*\nStatus: *${buildStatus}*\nAction: *${action}*"

    

    // Add error message only if status is failed 

    if(buildStatus == 'FAILED') {

        message = message + "\nError description: *${ciError}*"

    }

    message = message + "\n Build Report: <${env.RUN_DISPLAY_URL}|Jump To Console>\n${taskList}"



    slackSend(color: colorCode, message: message, channel: channelName)

}



def jiraTaskProgress(String stage = 'Development') {

    def transitionNumber = ''

    def getTaskNumberCommand = ''

    def slackMessage = '*** Jira Tasks *** \n'



    // Set transition number and Command according to the stage name

    if (stage == "Development") {

        transitionNumber = "101"

        getTaskNumberCommand = "git log --merges -n1 --format='%s' | grep -oP 'PX-[0-9]*' || echo ''"

    } else {

        transitionNumber = "211"

        def tagName = sh(script: "git tag --sort=-\"v:refname\" | head -n2 | grep -v ${env.TAG_NAME}", returnStdout: true).toString().replace('\n', '')

        getTaskNumberCommand = "git log --merges ${tagName}..HEAD --format='%s' | grep -oP 'PX-[0-9]*' | uniq || echo ''"

    }

    

    // Get the task number from merged pull request title

    def taskNumber = sh(script: "${getTaskNumberCommand}", returnStdout: true).toString()

    if (taskNumber != '') {

        taskList = taskNumber.split('\n')

        for (task in taskList) {

            if (stage != 'Staging') {

                jiraChangeTaskStatus(task, transitionNumber)

            }

            def title = jiraTaskTitle(task).toString().replace('\n', '')

            def link = "https://finlex.atlassian.net/browse/${task}"

            slackMessage += "*${task}:* <${link}|${title}> \n"

        }

        return slackMessage

    } else {

        return "Get Task Failed: *There is no valid task for this merged PR.*"

    }

}



def jiraTaskTitle(String taskNumber) {

    def taskTitle = ''

    // Get task title from the Jira API endpoint

    withCredentials([string(credentialsId: 'JiraAPIToken', variable: 'password')]) {

        taskTitle = sh(script: "curl -XGET https://finlex.atlassian.net/rest/api/3/issue/${taskNumber} -u it@finlex.de:${password} | jq -r '.fields.summary'", returnStdout: true).toString()

    }

    return taskTitle

}



def jiraChangeTaskStatus(String taskNumber, String transitionNumber) {

    // Change task status with Jira API endpoint according to the stage

    withCredentials([string(credentialsId: 'JiraAPIToken', variable: 'password')]) {

        sh "curl -XPOST https://finlex.atlassian.net/rest/api/3/issue/${taskNumber}/transitions -u 'it@finlex.de:${password}' --data '{\"transition\":{\"id\":\"${transitionNumber}\"}}'"

    }

}



def buildImage(String stage = 'dev', String serviceName) {

    def dockerTagName = 'dev'

    // Decided which branch should be build according to the environment

    if (stage == 'dev' || stage == 'test') {

        dockerTagName = stage

    } else {

        dockerTagName = env.TAG_NAME

    }

    sh '$(aws ecr get-login --no-include-email --registry-ids 474975247937 --region eu-central-1)'

    sh "docker build -t 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${dockerTagName} ."

    if (stage != 'test') {

        sh "docker push 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${dockerTagName}"

    }

}



def deployFeatures(String stage = 'dev', String serviceName, String serviceGroup) {

    def dockerTagName = 'dev'

    def zone = ''

    def cluster = ''



    // According to the environment set different varibale

    if (stage == 'dev') {

        dockerTagName = 'dev'

        zone = 'eu-west-1'

        cluster = 'dev0'

    } else if (stage == 'stage') {

        dockerTagName = env.TAG_NAME

        zone = 'eu-central-1'

        cluster = 'staging1'

    } else {

        dockerTagName = env.TAG_NAME

        zone = 'eu-central-1'

        cluster = 'prod'

    }



    sh '$(aws ecr get-login --no-include-email --registry-ids 474975247937 --region eu-central-1)'

    // if the environment is not development, we need to tagged version to the stage name

    if (stage != 'dev') {

        sh "docker pull 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${env.TAG_NAME}"

        sh "docker tag 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${env.TAG_NAME} 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${stage}"

        sh "docker push 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${stage}"

    }

    sh "docker run 474975247937.dkr.ecr.eu-central-1.amazonaws.com/${serviceName}:${env.TAG_NAME}

    

}
