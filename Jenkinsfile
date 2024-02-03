pipeline {
    agent { label 'Jenkins-Agent' }
    
    tools{
        jdk 'JDK'
        maven 'Maven'
        //sonarqubeScanner 'SonarQube' // Make sure 'SonarQube' matches the tool installation in Jenkins
  
       
        
    }
    
    environment{
        PATH = "/home/ubuntu/.nvm/versions/node/v18.18.2/bin:$PATH"
        DC_HOME= tool 'OWASP-DependencyCheck'
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "roopabs"
        DOCKER_PASS = 'dockerhub@7'
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
	    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {
        
        stage("Cleanup Workspace"){
                steps {
                cleanWs()
                }
        }
        
        stage('git-clone') {
            steps {
                echo 'git-clone'
                git branch: 'main', credentialsId: 'git_credential', url: 'https://github.com/Roopaabs/register-app.git'
            }
        }
        
        
        stage("Build Application"){
            steps {
                sh "mvn clean package"
            }

       }

       stage("Test Application"){
           steps {
                 sh "mvn test"
           }
       }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQube'
            
                    withSonarQubeEnv('SonarQube') {
                    sh "mvn clean verify sonar:sonar \
                     -Dsonar.projectKey=register-app \
                     -Dsonar.projectName='register-app' \
                     -Dsonar.host.url=http://52.22.3.214:9000 \
                     -Dsonar.token=squ_f90b4538b67c83fc8fe48a30264bafdea571b9e5"
                     }
                }
            }
        }
        
        stage("Quality Gate"){
           steps {
               script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-credential'
                }	
            }

        }
        
        
        stage('build') {
            steps {
                echo 'build the package'
                sh 'mvn package -DskipTests=true'
            }
        }
        
        
        
        stage('build and tag docker image'){
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'Docker') {
                
                        docker_image = docker.build "${IMAGE_NAME}"
                       
                    }
                    
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'Docker') 
                    {
                
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                        
                    }
                }
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                script {
                    try {
                        echo 'Running Trivy Docker image scan or file system scan'
                        sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image roopabs/register-app-pipeline:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table')
                    } 
                    catch (Exception e) {
                        echo "Error during Trivy scan: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE' // Mark the build as unstable on failure
                    }
                }
            }
        }
        
        stage ('Cleanup Artifacts') {
           steps {
               script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
               }
          }
       }
       
       stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-52-22-3-214.compute-1.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
                }
            }
       }
    
        
        
       
    }
}

