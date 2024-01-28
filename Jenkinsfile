// Properties
properties([
    buildDiscarder(
        logRotator(
            artifactDaysToKeepStr: "",
            artifactNumToKeepStr: "",
            daysToKeepStr: "",
            numToKeepStr: "5"
        )
    )
])

// Environment
podTemplate(
    cloud: "K8s Cluster 01",
    slaveConnectTimeout: 300,
    idleMinutes: 5,
    serviceAccount: "jenkins-admin",
    yaml: '''
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
                    - k8s-worker-3
          containers:
          - command:
            - cat
            image: rgyetvai/custom-dind:latest
            name: custom-dind
            resources:
              limits:
                cpu: "4"
                memory: 8Gi
              requests:
                cpu: "2"
                memory: 1Gi
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
) {
// Pipeline
    node(POD_LABEL) {
        try {
            stage('Test ENV VAR') {
                def SCANNER_HOME = tool 'sonar-scanner'
                echo "Scanner Home: ${SCANNER_HOME}"
            }
            stage('Checkout') {
                git url: 'git@github.com:renegyetvai/spring-petclinic.git', branch: 'parallelized-jenkinsfile', credentialsId: 'git_jenkins_ba_01'
            }
            stage('Prepare Pipeline Run') {
                def stages = [:]
                stages["Build Sources"] = {
                    container('custom-dind') {
                        sh 'mvn --version'
                        sh 'mvn clean -DskipTests'
                    }
                }
                stages["Setup Test Instance"] = {
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
                parallel(stages)
            }
            stage('Test & Scan Sources') {
                container('custom-dind') {
                    parallel getWrappedStages()
                }
            }
            stage('Docker Build') {
                container('custom-dind') {
                    sh 'docker run -d --name temp_container petclinic-micro-svc:latest'
                    sh 'docker commit temp_container rgyetvai/petclinic:latest'
                    sh 'docker rm -f temp_container'
                }
            }
            stage('Docker Push') {
                container('custom-dind') {
                    withEnv(['DOCKERHUB_CREDENTIALS = credentials(\'rgyetvai-dockerhub\')']) {
                        sh 'docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW'
                        sh 'docker push rgyetvai/petclinic:latest'
                    }
                }
            }
            echo "SUCCESS"
        } catch (err) {
            echo "FAILURE: \n${err}"
            throw err
        } finally {
            sh 'docker logout'
            container('custom-dind') {
                sh 'docker stop petclinic-test'
                sh 'docker rm -f petclinic-test'

                sh 'docker network rm -f zapnet'

                sh 'docker rmi -f rgyetvai/petclinic:testing'
                sh 'docker rmi -f softwaresecurityproject/zap-stable'
            }
            echo "Pipeline finished"
        }
    }  
}

// Functions
def getWrappedStages() {
    stages = [:]
    stages["Tests & SAST"] = {
        parallel nestedStagesOne()
    }
    stages["Prepare, Build & Scan"] = {
        parallel nestedStagesTwo()
    }
    return stages
}

def nestedStagesOne() {
    stages = [:]
    stages["Unit & Integration Tests"] = {
        stage('Unit & Integration Tests') {
            sh 'mvn test'
            sh 'mvn verify'
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
            def SCANNER_HOME = tool 'sonar-scanner'
            echo "Scanner Home B1: ${SCANNER_HOME}"

            withSonarQubeEnv('sonarqube') {
                echo "Scanner Home B2: ${SCANNER_HOME}"
                sh '${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectName=petclinic-example -Dsonar.java.binaries=. -Dsonar.projectKey=petclinic-example -Dsonar.exclusions=dependency-check-report.html'
            }
        }
    }
    return stages
}

def nestedStagesTwo() {
    stages = [:]
    stages["Trivy Image Scan"] = {
        stage('Trivy Image Scan') {
            // Severity levels: MEDIUM,HIGH,CRITICAL
            sh 'trivy image --exit-code 1 --severity CRITICAL rgyetvai/petclinic:testing'
        }
    }
    stages["OWASP ZAP Scan"] = {
        stage('OWASP ZAP Scan') {
            sh 'docker pull softwaresecurityproject/zap-stable'
            sh 'docker run --net zapnet --user root -v $(pwd):/zap/wrk/:rw -t softwaresecurityproject/zap-stable zap-baseline.py -t https://172.16.0.2:8080 -g gen.conf -r report.html -I'
        }
    }
    stages["DAST 02"] = {
        stage('DAST 02') {
            echo "Executing DAST 02"
        }
    }
    return stages
}
