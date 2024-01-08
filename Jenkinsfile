def registry = 'https://coder9a.jfrog.io/'
def imageName = 'coder9a.jfrog.io/cicd-docker-docker-local/ttrend'
def version   = '2.1.3'
pipeline {
    agent any
    environment {
        PATH = "/opt/apache-maven-3.9.4/bin:$PATH"
    }
    stages {
        stage("build"){
            steps {
                echo "--------- build started ----------"
                sh 'mvn clean deploy -Dmaven.test.skiip=true'
                echo "--------- build completed ----------"
                // sh 'mvn clean install -U'
            }
        }
        stage ("test") {
            steps {
                echo "--------- unit test started ----------"
                sh 'mvn surefire-report:report'
                echo "--------- unit test completed ----------"
            }
        }
        stage ('SonarQube analysis') {
            environment {
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        stage ("Quality Gate") {
            steps {
                script {
                    timeout(time:1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        stage("Jar Publish") {
            steps {
                script {
                        echo '<--------------- Jar Publish Started --------------->'
                        def server = Artifactory.newServer url:registry+"/artifactory" ,  credentialsId:"artifactory-token"
                        def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                        def uploadSpec = """{
                            "files": [
                                {
                                "pattern": "jarstaging/(*)",
                                "target": "CICD-maven/{1}",
                                "flat": "false",
                                "props" : "${properties}",
                                "exclusions": [ "*.sha1", "*.md5"]
                                }
                            ]
                        }"""
                        def buildInfo = server.upload(uploadSpec)
                        buildInfo.env.collect()
                        server.publishBuildInfo(buildInfo)
                        echo '<--------------- Jar Publish Ended --------------->'  
                
                    }
                }   
            } 
        stage(" Docker Build ") {
            steps {
                script {
                echo '<--------------- Docker Build Started --------------->'
                app = docker.build(imageName+":"+version)
                echo '<--------------- Docker Build Ends --------------->'
                }
            }
        }

        stage (" Docker Publish "){
            steps {
                script {
                echo '<--------------- Docker Publish Started --------------->'  
                    docker.withRegistry(registry, 'artifactory-token'){
                        app.push()
                    }    
                echo '<--------------- Docker Publish Ended --------------->'  
                }
            }
        }
        stage(" Deploy via Helm Chart") {
            steps {
                script {
                    echo '<--------------- Helm Deploy Started --------------->'
                    sh 'helm install ttrend ttrend-0.1.0.tgz --namespace concourse --create-namespace --wait'
                    echo '<--------------- Helm deploy Ends --------------->'
                }
            }
        }
    }
}