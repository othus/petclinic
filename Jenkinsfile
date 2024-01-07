pipeline {
    agent any
    environment {
        NEXUS_REPO = credentials('nexus-repo-url')
    }
    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 4, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build Artifact') {
            steps {
                //sh 'mvn clean install -DskipTests'
                sh 'mvn install -DskipTests'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }
        stage('Nexus Login') {
            steps{
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-user-password', passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {
                        sh 'echo $PASSWORD | docker login --username $USER --password-stdin http://$NEXUS_REPO'
                        //sh 'docker push $NEXUS_REPO/myapp:latest'
                    }
                }
            }
        }    
        stage('Push to Nexus Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/myapp:latest'
                //sh 'docker push 44.212.5.145/:8085/myapp:latest'
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@10.0.1.237 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook -i /etc/ansible/stage-hosts stage-env-playbook.yml"'
                }
            }
        }
        //stage('slack notification') {
        //    steps {
        //        slackSend channel: 'jenkins-alert',
        //        message: 'App deployed to Stage, needs approval to deploy to prod',
        //        teamDomain: 'paceu1',
        //        tokenCredentialId: 'slack-credentials'
        //    }
        //}
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@10.0.1.237 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook -i /etc/ansible/prod-hosts prod-env-playbook.yml"'
                }
            }
        }
    }
}
