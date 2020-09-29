pipeline {
    agent {
        kubernetes {
            label 'mosaic-ci-pod'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven-sumo
    image: maven:3.6.3-adoptopenjdk-8
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: "/home/jenkins/.m2/settings.xml"
      name: "settings-xml"
      readOnly: true
      subPath: "settings.xml"
    - name: m2-repo
      mountPath: /home/jenkins/.m2/repository
    resources:
      limits:
        memory: "2Gi"
        cpu: "1"
      requests:
        memory: "2Gi"
        cpu: "1"    
  - name: jnlp
    image: 'eclipsecbijenkins/basic-agent:3.35'
    volumeMounts:
    - mountPath: "/home/jenkins/.m2/settings-security.xml"
      name: "settings-security-xml"
      readOnly: true
      subPath: "settings-security.xml"
    - mountPath: "/home/jenkins/.m2/settings.xml"
      name: "settings-xml"
      readOnly: true
      subPath: "settings.xml"
    - name: m2-repo
      mountPath: /home/jenkins/.m2/repository
    - mountPath: "/opt/tools"
      name: "volume-0"
      readOnly: false
    resources:
      limits:
        memory: "2Gi"
        cpu: "1"
      requests:
        memory: "2Gi"
        cpu: "1"
  volumes:
  - name: "settings-security-xml"
    secret:
      items:
      - key: "settings-security.xml"
        path: "settings-security.xml"
      secretName: "m2-secret-dir"
  - name: "settings-xml"
    secret:
      items:
      - key: "settings.xml"
        path: "settings.xml"
      secretName: "m2-secret-dir"      
  - name: m2-repo
    emptyDir: {}
  - name: "volume-0"
    persistentVolumeClaim:
      claimName: "tools-claim-jiro-mosaic"
      readOnly: true
"""
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven-sumo') {
                    sh 'mvn clean install -DskipTests -fae -T 4'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven-sumo') {
                    sh 'echo "skip"'//sh 'mvn test -fae -T 4 -P coverage'
                }
            }

            post {
                always {
                    sh 'echo "skip"'//junit '**/surefire-reports/*.xml'
                }
            }
        }

        stage('Integration Tests') {
            steps {
                container('maven-sumo') {
                    sh 'echo "skip"'//sh 'mvn test -fae -P integration-tests,coverage'
                }
            }

            post {
                always {
                    sh 'echo "skip"'//junit 'test/**/surefire-reports/*.xml'
                }
            }
        }

        stage('Analysis') {
            steps {
                container('maven-sumo') {
                    sh 'echo "skip"'//sh 'mvn site -T 4'
                }
            }

            post {
                always {
                    sh 'echo "skip"'
//                    jacoco exclusionPattern: '**/ClientServerChannelProtos*.class', skipCopyOfSrcFiles: true, sourceExclusionPattern: '**/*.*', sourceInclusionPattern: '', sourcePattern: 'x'
//                    recordIssues(sourceCodeEncoding: 'UTF-8', tools: [
//                            spotBugs(),
//                            checkStyle(),
//                            taskScanner(highTags: 'FIXME', normalTags: 'TODO', ignoreCase: true, includePattern: '**/*.java')
//                    ])
                }
            }
        }

//        stage('Deploy') {
//            when {
//                expression { env.BRANCH_NAME == 'main' }
//            }
//            steps {
//                container('maven-sumo') {
//                    sh 'mvn deploy -DskipTests'
//                }
//            }
//        }

        stage ('Deploy') {
            when {
                branch 'main'
            }
            steps {
                container('jnlp') {
                    sh '/opt/tools/apache-maven/3.6.3/bin/mvn deploy -DskipTests -Dmaven.repo.local=/home/jenkins/.m2/repository'
                }
            }
        }
    }

    post {
        success {
            cleanWs()
        }
        fixed {
            sendMail("Successful")
        }
        unstable {
            sendMail("Unstable")
        }
        failure {
            sendMail("Failed")
        }
    }
}

def sendMail(status) {
    emailext(
            recipientProviders: [culprits()],
            subject: status + ": ${currentBuild.fullDisplayName}",
            body: '${JELLY_SCRIPT, template="html-with-health-and-console"}',
            mimeType: 'text/html'
    )
}