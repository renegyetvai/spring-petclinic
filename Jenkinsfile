pipeline {
    agent {
        kubernetes {
            cloud 'K8s Cluster 01'
            defaultContainer 'custom-dind'
            inheritFrom "dev-agent-${env.BUILD_NUMBER}"
            slaveConnectTimeout 300
            idleMinutes 5
            serviceAccount 'jenkins-admin'
            yaml'''
                apiVersion: v1
                kind: Pod
                metadata:
                  name: dev-agent-${env.BUILD_NUMBER}
                  namespace: devops-tools
                spec:
                    affinity:
                        nodeAffinity:
                            requiredDuringSchedulingIgnoredDuringExecution:
                                nodeSelectorTerms:
                                - matchExpressions:
                                    - key: kubernetes.io/hostname
                                      operator: In
                                      values:
                                      - k8s-worker-3
                    containers:
                    - name: custom-dind
                      image: rgyetvai/custom-dind:latest
                      imagePullPolicy: Always
                      command:
                      - cat
                      tty: true
                      resources:
                        limits:
                          memory: "8Gi"
                          cpu: "4"
                        requests:
                          memory: "1Gi"
                          cpu: "1"
                      volumeMounts:
                      - name: docker-sock-volume
                        mountPath: /var/run/docker.sock
                    volumes:
                    - name: docker-sock-volume
                      hostPath:
                        path: /var/run/docker.sock
                        type: Socket
                '''
        }
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        parallelsAlwaysFailFast()
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_CREDENTIALS = credentials('rgyetvai-dockerhub')
    }
    stages {
        stage('Git Checkout') {
            steps {
                container('custom-dind') {
                    script {
                        git branch: 'parallelized-jenkinsfile',
                            credentialsId: 'git_jenkins_ba_01',
                            url: 'git@github.com:renegyetvai/spring-petclinic.git'
                    }
                }
            }
        }
        stage('Compile Sources') {
            steps {
                container('custom-dind') {
                    sh 'mvn --version'
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        stage('Setup Test Instance') {
            steps {
                container('custom-dind') {
                    sh 'docker rm -f petclinic-test'

                    sh 'docker network rm -f zapnet'
                    sh 'docker network create --driver=bridge --subnet=172.16.0.0/24 zapnet'

                    sh 'mvn spring-boot:build-image -D spring-boot.build-image.imageName=petclinic-micro-svc -DskipTests'
                    sh 'docker run -d --name temp_container petclinic-micro-svc:latest'
                    sh 'docker commit temp_container rgyetvai/petclinic:testing'
                    sh 'docker rm -f temp_container'

                    sh 'docker rm -f petclinic-test'
                    sh 'docker run -d --name petclinic-test --net zapnet --ip 172.16.0.2 -p 8080:8080 rgyetvai/petclinic:testing'
                }
            }
        }
        stage('Test & Scan Sources') {
            steps {
                container('custom-dind') {
                    script {
                        parallel getWrappedStages()
                    }
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('custom-dind') {
                    script {
                        sh 'docker run -d --name temp_container petclinic-micro-svc:latest'
                        sh 'docker commit temp_container rgyetvai/petclinic:latest'
                        sh 'docker rm -f temp_container'
                    }
                }
            }
        }
        stage('Docker Push') {
            steps {
                container('custom-dind') {
                    script {
                        sh 'docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW'
                        sh 'docker push rgyetvai/petclinic:latest'
                    }
                }
            }
        }
    }
    post {
        always {
            sh 'docker logout'
            container('custom-dind') {
                sh 'docker stop petclinic-test'
                sh 'docker rm -f petclinic-test'

                sh 'docker network rm -f zapnet'

                sh 'docker rmi -f rgyetvai/petclinic:testing'
                sh 'docker rmi -f softwaresecurityproject/zap-stable'
            }
        }
        success {
            echo 'Build Success'
        }
        failure {
            echo 'Build Failed'
        }
        unstable {
            echo 'Build Unstable'
        }
        changed {
            echo 'Build Changed'
        }
    }
}

def getWrappedStages() {
    stages = [:]
    stages["Tests & SAST"] = {
        stage('Tests & SAST') {
            sh """
                echo "Executing Tests & SAST"
            """
        }
        parallel nestedStagesOne()
    }
    stages["Prepare, Build & Scan"] = {
        stage('Prepare DAST') {
            sh """
                echo "Executing DAST Preparation"
            """
        }
        parallel nestedStagesTwo()
    }
    return stages
}

def nestedStagesOne() {
    stages = [:]
    stages["Tests & SAST Scans"] = {
        stage('Unit & Integration Tests') {
            sh 'mvn test'
            sh 'mvn verify'
        }
        stage('OWASP Dependency Scan') {
            dependencyCheck additionalArguments: '', odcInstallation: 'DP-check'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        }
        stage('SonarQube Scan') {
            withSonarQubeEnv('sonarqube') {
                sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=petclinic-example \
                -Dsonar.java.binaries=. \
                -Dsonar.projectKey=petclinic-example \
                -Dsonar.exclusions=dependency-check-report.html '''
            }
        }
    }
    return stages
}

def nestedStagesTwo() {
    stages = [:]
    stages["Container & DAST Scans"] = {
        stage('Trivy Image Scan') {
            // Severity levels: MEDIUM,HIGH,CRITICAL
            sh 'trivy image --exit-code 1 --severity CRITICAL rgyetvai/petclinic:testing'
        }
        stage('OWASP ZAP Scan') {
            sh 'docker pull softwaresecurityproject/zap-stable'
            sh 'docker run --net zapnet --user root -v $(pwd):/zap/wrk/:rw -t softwaresecurityproject/zap-stable zap-baseline.py -t https://172.16.0.2:8080 -g gen.conf -r report.html -I'
        }
        stage('DAST 02') {
            sh """
                echo "Executing DAST 02"
            """
        }
    }
    return stages
}
