pipeline {
  agent any

  triggers {
      pollSCM('* * * * *') // Runs every minute
  }  
  
  environment {
    BASTION_INSTANCE_ID = credentials('bastion-id')
    ANSIBLE_IP = credentials('ansible-ip')  // Keeping IP since we need it for port 22 connection
    AWS_REGION = 'us-east-1'
    APP_REPO_NAME = "Sock-Shop-App-Repo"
    PORT = "30001"
    DEPLOYMENT_MANIFEST = "complete.yaml"
    GIT_REPO_URL = "https://github.com/CloudHight/Sock-Shop-App-Repo.git"
    STAGE_BRANCH = "stage"
    MAIN_BRANCH = "main"
  }
  
  stages {
    stage ('Deploying to Stage Environment') {
      steps {
          script {
            // Start SSM session to bastion with port forwarding for SSH (port 22)
            sh '''
              aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                &
              sleep 5  # Wait for port forwarding to establish
            '''
            
            // SSH through the tunnel to Ansible server on port 22
            sshagent(['bastion-key', 'ansible-key']) {
              sh '''
                ssh -o StrictHostKeyChecking=no \
                    -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                    ubuntu@${ANSIBLE_IP} \
                    "ansible-playbook /etc/ansible/playbooks/test.yml"
              '''
            }
            
            // Terminate the SSM session
            sh 'pkill -f "aws ssm start-session"'
          }
      }
    }
    
    stage ('Slack Notification for prod') {
      steps {
        slackSend channel: 'Cloudhight', message: 'New Stage Deployment', teamDomain: '#1st-december-sock-shop-kubernetes-project-using-ansible', tokenCredentialId: 'slack'
      }
    }
    
    stage ('DAST Scan') {
      steps {
        sh '''
          chmod 777 $(pwd)
          docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://stage.work-experience-2025.buzz -g gen.conf -r testreport.html || true
        '''
      }
    }
    
    stage ('Prompt for Approval') {
      steps {
        timeout(activity: true, time: 5) {
          input message: 'Review before approval', submitter: 'admin'
        }
      }
    }

    stage('Update Deployment Manifest in Stage Branch') {
        steps {
            script {
                withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                        rm -rf ${APP_REPO_NAME} || true
                        git clone ${GIT_REPO_URL}
                        cd ${APP_REPO_NAME}
                        git config --global user.email "jenkins@set30.space"
                        git config --global user.name "Jenkins CI"
                        git checkout ${STAGE_BRANCH}
                        git pull origin ${STAGE_BRANCH} --rebase
                        sed -i 's|nodeport: ${PORT}:.*|nodeport: ${PORT}|' ${DEPLOYMENT_MANIFEST}
                        git add ${DEPLOYMENT_MANIFEST}
                        if git diff --cached --quiet; then
                            echo "No changes detected, skipping commit in stage branch."
                        else
                            git commit -m "Update nodeport number to ${PORT} in stage branch"
                            git push https://${GIT_USERNAME}:${GIT_TOKEN}@${GIT_REPO_URL.replace('https://', '')} ${STAGE_BRANCH}
                        fi
                    """
                }
            }
        }
    }
    
    stage ('Merge stage into main') {
      steps {
        script {
          // Merge `stage` branch into `main` using the existing git-cred credentials
          withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              set -e
              cd ${APP_REPO_NAME}
              git config --global user.email "jenkins@set30.space"
              git config --global user.name "Jenkins CI"
              git fetch origin
              git checkout ${MAIN_BRANCH}
              git pull origin ${MAIN_BRANCH}
              git merge --no-ff -X theirs origin/${STAGE_BRANCH} -m "Automated merge of stage into main by Jenkins \${BUILD_NUMBER}"
              REPO_URL_HTTPS=\$(echo "${GIT_REPO_URL}" | sed 's|https://||')
              git push https://\${GIT_USERNAME}:\${GIT_TOKEN}@\${REPO_URL_HTTPS} ${MAIN_BRANCH}
            """
          }
        }
      }
    }
    
    stage ('Deploying to Prod Environment') {
      steps {
          script {
            sh '''
              aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                &
              sleep 5
            '''
            
            sshagent(['bastion-key', 'ansible-key']) {
              sh '''
                ssh -o StrictHostKeyChecking=no \
                    -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                    ubuntu@${ANSIBLE_IP} \
                    "ansible-playbook /etc/ansible/playbooks/test2.yml"
              '''
            }
            
            sh 'pkill -f "aws ssm start-session"'
          }
      }
    }
    
    stage ('Slack Notification') {
      steps {
        slackSend channel: 'Cloudhight', message: 'New Production Deployment', teamDomain: '1st-december-sock-shop-kubernetes-project-using-ansible', tokenCredentialId: 'slack'
      }
    }
  }
}
