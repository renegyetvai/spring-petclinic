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
                  name: jenkins-build-pod
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
                    name: custom-dind
                    image: rgyetvai/custom-dind:latest
                    resources:
                      limits:
                        cpu: "2"
                        memory: 4Gi
                      requests:
                        cpu: "1"
                        memory: 2Gi
                    tty: true
                    volumeMounts:
                    - mountPath: /var/run/docker.sock
                      name: docker-sock-volume
                  volumes:
                  - hostPath:
                      path: /var/run/docker.sock
                      type: Socket
                    name: docker-sock-volume
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
        stage('Docker Login') {
            steps {
                container('custom-dind') {
                    script {
                        sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    }
                }
            }
        }
        stage('Docker Scout') {
            steps {
                container('custom-dind') {
                    // Install Docker Scout
                    sh 'curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s -- -b /usr/local/bin'

                    // Analyze and fail on critical or high vulnerabilities
                    sh 'docker-scout cves $IMAGE_TAG_TEST --exit-code --only-severity critical'
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
        stage('Unit & Integration Tests') {
            steps {
                container('custom-dind') {
                    sh 'mvn test'
                    //sh 'mvn verify'
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
        stage('OWASP Dependency Scan') {
            steps {
                container('custom-dind') {
                    // API Key: --nvdApiKey 'aa4003c1-eb27-42a5-aba8-af1e7acdb7d5'
                    dependencyCheck additionalArguments: '', odcInstallation: 'DP-check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=petclinic-example \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=petclinic-example \
                    -Dsonar.exclusions=dependency-check-report.html '''
                }
            }
        }
        //stage('Busy Waiting Simulation') {
        //    steps {
        //        container('custom-dind') {
        //            sh 'sleep 300'
        //        }
        //    }
        //}
        //stage('Trivy Image Scan') {
        //    steps {
        //        container('custom-dind') {
                    // Severity levels: MEDIUM,HIGH,CRITICAL
        //            sh 'trivy image --exit-code 1 --severity CRITICAL --scanners vuln --reset rgyetvai/petclinic:testing'
        //        }
        //    }
        //}
        stage('OWASP ZAP Scan') {
            steps {
                container('custom-dind') {
                    sh 'docker pull softwaresecurityproject/zap-stable'
                    sh 'docker run --net zapnet --name zap --user root -v $(pwd):/zap/wrk/:rw -t softwaresecurityproject/zap-stable zap-full-scan.py -t https://172.16.0.2:8080 -g gen.conf -r report.html -I'
                }
            }
        }
        stage('Nikto Scan') {
            steps {
                container('custom-dind') {
                    sh 'docker run --net zapnet --name nikto --rm frapsoft/nikto -h https://172.16.0.2:8080'
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
                        sh 'docker push rgyetvai/petclinic:latest'
                    }
                }
            }
        }
    }
    post {
        always {
            container('custom-dind') {
                sh 'docker logout'

                sh 'docker stop petclinic-test'
                sh 'docker rm -f petclinic-test'

                sh 'docker rm -f zap'
                sh 'docker rm -f nikto'

                sh 'docker network rm -f zapnet'

                sh 'docker rmi -f rgyetvai/petclinic:testing'
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
