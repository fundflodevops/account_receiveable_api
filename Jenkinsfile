def call(Map params) {
    // Define defaults
    def DEPLOY_BRANCH = 'main'
    def VERSION_NO = ""
    def BASE_VERSION = '0.0.2'
    def BUCKET_NAME ="test-1";
    
    pipeline {
        agent any
        options {
            disableConcurrentBuilds()
            timeout(time: 1, unit: 'HOURS')
            buildDiscarder(logRotator(numToKeepStr: '3'))
        }
        stages {
            stage('CREATE & PUSH TAG') {
                when {
                    expression { env.BRANCH_NAME == DEPLOY_BRANCH }
                }
                steps {
                    script {
                        VERSION_NO = gitTag("${BASE_VERSION}")
                        GENERATED_ID = VERSION_NO.replace(".", "")
                    }
                }
            }
            stage('UPDATE ENVIRONMENT FILE') {
                when {
                    expression { env.BRANCH_NAME == DEPLOY_BRANCH }
                }
                steps {
                    script {
                        sh "sed -e 's/%GENERATED_ID%/${GENERATED_ID}/g;s/%API_URL%/${params.API_URL}/g;s/%BUCKET_NAME%/${params.BUCKET_NAME}/g;' src/environments/environment_generate.ts > src/environments/environment.prod.ts"
                        sh "sed -e 's/%GENERATED_ID%/${GENERATED_ID}/g;s/%API_URL%/${params.API_URL}/g;s/%BUCKET_NAME%/${params.BUCKET_NAME}/g;' src/environments/environment_generate.ts > src/environments/environment.ts"
                    }
                }
            }
        }
    }
}
