//The Jenkinsfile is shared as a separate file along along with the Question Paper. You can use it for copying content. 
//Note: Be careful of spacesand new line characterswhile editing Jenkinsfileand running commands
import groovy.json.JsonSlurper
node{
    def BUILD_NUMBER= env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
    def SFDC_USERNAMEdef HUB_ORG=env.HUB_ORG_DH
    def SFDC_HOST=env.SFDC_HOST_DH
    def JWT_KEY_CRED_ID=env.JWT_CRED_ID_DH
    def CONNECTED_APP_CONSUMER_KEY=env.CONNECTED_APP_CONSUMER_KEY_DH
    def toolbelt= tool 'toolbelt'
    
    println 'All environment variables to be printed here..'
    println BUILD_NUMBER
    println RUN_ARTIFACT_DIR
    println SFDC_USERNAME
    println HUB_ORG
    println SFDC_HOST
    println JWT_KEY_CRED_ID
    println CONNECTED_APP_CONSUMER_KEY

    stage('checkout source'){
        checkout scm
        print "Checkout done successfully..."
    }

    withCredentials([file(credentialsId:JWT_KEY_CRED_ID, variable:'jwt_key_file')])
    {
        print "JWT Key Credential Id Fetched successfully"
        stage('Create Scratch Org') {
            print "connecting to Salesforce..."
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
            if (rc != 0) { error 'hub org authorization failed' }
            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:org:create -f config/project-scratch-def.json -a ebikes --setdefaultusername --json"
            printf rmsg
            def jsonSlurper = new JsonSlurper()
            def robj = jsonSlurper.parseText(rmsg)
            if (robj.status!= 0) { error 'org creation failed: ' + robj.message }

            SFDC_USERNAME=robj.result.username
            robj = null
            print "scratch org created with username ${SFDC_USERNAME}..."
        }

        stage('Push To Test Org') {
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:source:push --targetusername ${SFDC_USERNAME}"
            if (rc != 0) {error 'push all failed...'}
            print "push all completed.."
            // assign permset
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname ebikes"
            if (rc != 0) {error 'push permission set failed...'}
            print "push permission set completed..."
        }

        stage('Run Apex Test') {
            sh "mkdir -p ${RUN_ARTIFACT_DIR}"
            timeout(time: 120, unit: 'SECONDS') {
                rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:apex:test:run --testlevel RunLocalTests --outputdir ${RUN_ARTIFACT_DIR} --resultformat json --targetusername ${SFDC_USERNAME}"
                if (rc != 0) {error 'apex test run failed...'}
            }
            print "apex test run completed..."
        }
        
        stage('Collect Results') {
            junit keepLongStdio: true, testResults: 'tests/**/*-junit.xml'
            print "Test results collected successfully"
        }
        
        stage('Delete Test Org') {
            timeout(time: 120, unit: 'SECONDS') {
                rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:org:delete --targetusername ${SFDC_USERNAME} --noprompt"
                if (rc != 0) {error 'org deletion request failed'}
            }
            print "Org delete request completed"
        }
    }
}