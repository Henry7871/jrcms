/**
* Requirement:
* 1. Kubernetes cluster
* 2. Credentials: 
*    - docker-hub
*    - aws-eb-key
* 3. AWS Elastic Benstalk in us-east-1 region
*    - Application: jrcms
*    - Environment: jrcms-test, jrcms-staging and jrcms-production
**/

podTemplate(
        containers: [containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
                     containerTemplate(name: 'eb', image: 'mini/eb-cli', command: 'cat', ttyEnabled: true)],
        volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
  ) {

  node(POD_LABEL) {
      
    def image
    withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASSWD')]) {
        image = "${USER}/jrcms:${currentBuild.number}"
    }
    
    stage('Build and Test') {
        checkout scm
        container('docker') {
            sh "docker build -t ${image} ."
        }
    }
    
    if (env.BRANCH_NAME == 'master') {
        stage('Push Docker image') {
                        // docker_username = credentials('docker-hub')
            // username = docker_username password=%SECRET_VALUE%
            withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASSWD')]) {
            container('docker') {
                sh "docker login --username ${USER} --password ${PASSWD}"
                sh "docker push ${image}"
            }
            }
        }
        
        stage("Deploy to test environment") {
            deployToEB('test')
        }
        
        stage("Integration test to test environment") {  
            smokeTest('test')
        }
        
        stage("Deploy to staging environment") {
            deployToEB('staging')
        }
        
        stage("Integration test to staging environment") {  
            smokeTest('staging')
        }
        
        stage("Deploy to production environment") {
            deployToEB('prod')
        }
    }
  }
}

def smokeTest(environment) {     
    container('eb') {
        String  test_url = "http://jrcmshenry-${environment}.eba-grs4pgnh.us-east-2.elasticbeanstalk.com/"
        int status = sh(script:"curl -sLI -w '%{http_code}\n' $test_url -o /dev/null", returnStdout: true)
        if (status != 200 &&  status != 201){
            error(
                "Return status code = $status when calling $test_url"
            )
        }
   }
}

def deployToEB(environment) {
    checkout scm
    withCredentials([usernamePassword(credentialsId: 'aws-eb-jenkins', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            container('eb') {
                withEnv(["AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}", "AWS_REGION=us-east-2"]) {
                    dir("deployment") {
                    sh "sh generate-dockerrun.sh ${currentBuild.number}"
                    sh "eb deploy jrcmshenry-${environment} -l ${currentBuild.number}"
                    }
                }
            }
    }
}
