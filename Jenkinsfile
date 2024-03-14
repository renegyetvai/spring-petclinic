pipeline {
    agent {
        kubernetes {
            cloud 'K8s Cluster 01'
            slaveConnectTimeout 300
            idleMinutes 5
            yaml'''
                apiVersion: v1
                kind: Pod
                metadata:
                  name: custom-build-pod
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
                            - k8s-worker-4
                  containers:
                  - command:
                    - cat
                    name: custom-dind-01
                    image: rgyetvai/custom-dind:latest
                    resources:
                      limits:
                        cpu: "2"
                        memory: 2Gi
                      requests:
                        cpu: 500m
                        memory: 500Mi
                    tty: true
                    volumeMounts:
                    - mountPath: /var/run/docker.sock
                      name: docker-sock-volume
                    - mountPath: /usr/share/git
                      name: shared-workspace
                  - command:
                    - cat
                    name: custom-dind-02
                    image: rgyetvai/custom-dind:latest
                    resources:
                      limits:
                        cpu: "2"
                        memory: 2Gi
                      requests:
                        cpu: 500m
                        memory: 500Mi
                    tty: true
                    volumeMounts:
                    - mountPath: /var/run/docker.sock
                      name: docker-sock-volume
                    - mountPath: /usr/share/git
                      name: shared-workspace
                  volumes:
                  - hostPath:
                      path: /var/run/docker.sock
                      type: Socket
                    name: docker-sock-volume
                  - name: shared-workspace
                    emptyDir: {}
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
        IMAGE_TAG_TEST = 'rgyetvai/petclinic:testing'
        IMAGE_TAG_LATEST = 'rgyetvai/petclinic:latest'
    }
    stages {
        stage('Prepare Workspace') {
            parallel {
                stage('Git Checkout') {
                    steps {
                        container('custom-dind-01') {
                            script {
                                git branch: 'parallelized-jenkinsfile',
                                    credentialsId: 'git_jenkins_ba_01',
                                    url: 'git@github.com:renegyetvai/spring-petclinic.git'
                                // copy the git repo to the shared workspace
                                sh 'cp -r ./* /usr/share/git'
                            }
                        }
                    }
                }
                stage('Docker Login') {
                    steps {
                        container('custom-dind-02') {
                            script {
                                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                            }
                        }
                    }
                }
            }
        }
        stage('Prepare Environment') {
            parallel {
                stage('Compile Sources') {
                    steps {
                        container('custom-dind-01') {
                            sh 'mvn --version'
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
                stage('Setup Test Instance') {
                    steps {
                        container('custom-dind-02') {
                            // Get shared workspace
                            sh 'cp -r /usr/share/git/* .'

                            sh 'docker rm -f petclinic-test'

                            sh 'docker network rm -f zapnet'
                            sh 'docker network create --driver=bridge --subnet=172.16.0.0/24 zapnet'

                            sh 'mvn spring-boot:build-image -D spring-boot.build-image.imageName=petclinic-micro-svc -DskipTests'
                            sh 'docker run -d --name temp_container petclinic-micro-svc:latest'
                            sh 'docker commit temp_container $IMAGE_TAG_TEST'
                            sh 'docker rm -f temp_container'

                            sh 'docker rm -f petclinic-test'
                            sh 'docker run -d --name petclinic-test --net zapnet --ip 172.16.0.2 -p 8080:8080 $IMAGE_TAG_TEST'
                        }
                    }
                }
            }
        }
        stage('Test & Scan Sources') {
            steps {
                script {
                    getWrappedStages()
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('custom-dind-02') {
                    script {
                        sh 'docker run -d --name temp_container petclinic-micro-svc:latest'
                        sh 'docker commit temp_container $IMAGE_TAG_LATEST'
                        sh 'docker rm -f temp_container'
                    }
                }
            }
        }
        stage('Docker Push') {
            steps {
                container('custom-dind-02') {
                    script {
                        sh 'docker push $IMAGE_TAG_LATEST'
                    }
                }
            }
        }
    }
    post {
        always {
            container('custom-dind-02') {
                sh 'docker logout'

                sh 'docker stop petclinic-test'
                sh 'docker rm -f petclinic-test'

                sh 'docker rm -f zap'
                sh 'docker rm -f nikto'

                sh 'docker network rm -f zapnet'

                sh 'docker rmi -f $IMAGE_TAG_TEST'
                sh 'docker rmi -f softwaresecurityproject/zap-stable'
                sh 'docker rmi -f frapsoft/nikto'
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
        stage('SAST') {
            container('custom-dind-01') {
                stage('Tests & SAST') {
                    sh """
                        echo "Executing Tests & SAST"
                    """
                }
                parallel nestedStagesOne()
            }
        }
    }
    stages["Snyk Scan"] = {
        // Snyks Jenkins plugin is not compatible with the container step
        stage('Snyk Scan') {
            snykSecurity severity: 'critical', snykInstallation: 'snyk@latest', snykTokenId: 'renegyetvai-snyk-api-token'
        }
    }
    stages["Prepare, Build & Scan"] = {
        stage('DAST') {
            container('custom-dind-02') {
                stage('Prepare, Build & Scan') {
                    sh """
                        echo "Executing Prepare, Build & Scan"
                    """
                }
                parallel nestedStagesTwo()
            }
        }
    }
    parallel stages
}

def nestedStagesOne() {
    stages = [:]
    stages["Unit & Integration Tests"] = {
        stage('Unit & Integration Tests') {
            sh 'mvn test'
        }
    }
    stages["OWASP Dependency Scan"] = {
        stage('OWASP Dependency Scan') {
            dependencyCheck additionalArguments: '', odcInstallation: 'DP-check'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        }
    }
    stages["SonarQube Scan"] = {
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
    //stages["Trivy Image Scan"] = {
    //    stage('Trivy Image Scan') {
            // Severity levels: MEDIUM,HIGH,CRITICAL
    //        sh 'trivy image --exit-code 1 --severity CRITICAL --scanners vuln --reset $IMAGE_TAG_TEST'
    //    }
    //}
    stages["OWASP ZAP Scan"] = {
        stage('OWASP ZAP Scan') {
            sh 'docker pull softwaresecurityproject/zap-stable'
            sh 'docker run --net zapnet --name zap --user root -v $(pwd):/zap/wrk/:rw -t softwaresecurityproject/zap-stable zap-full-scan.py -t https://172.16.0.2:8080 -g gen.conf -r report.html -I'
        }
    }
    stages["Nikto Scan"] = {
        stage('Nikto Scan') {
            //sh 'docker run --net zapnet --name nikto --rm frapsoft/nikto -h https://172.16.0.2:8080'
            sh 'docker run --net zapnet --name nikto --rm frapsoft/nikto -h 172.16.0.2 -p 8080'
        }
    }
    stages["Docker Scout"] = {
        stage('Docker Scout') {
            // Install Docker Scout
            sh 'curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s -- -b /usr/local/bin'

            // Analyze and fail on critical or high vulnerabilities
            sh 'docker-scout cves $IMAGE_TAG_TEST --exit-code --only-severity critical,high'
        }
    }
    return stages
}
