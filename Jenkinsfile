pipeline {
    agent any
    environment {
        IMAGE_NAME = "mz-react-todo"
        IMAGE_TAG = "latest"
        NEXUS_URL = "nginx-nexus.connectme365.com/hosted"
        DOCKER_SWARM_MANAGER = "jenkins@10.3.31.84"
        NEXUS_CREDENTIALS_ID = "jenkins-mz-rd"
        SSH_CREDENTIALS_ID = "publish_via_ssh"
    }
    stages {
        stage('Clone Repo'){
            steps{
                cleanWs() // Clean up workspace
                echo "Cloning source code ..."
                /*
                Trid using 'checkout scm' but it returned the following error message:
                org.codehaus.groovy.control.MultipleCompilationErrorsException: startup failed:
                WorkflowScript: 20: unexpected token: @ @ line 20, column 151.
                temservices-github', url: 'git@github.co
                */
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'systemservices-github', url: 'git@github.com:mcl-michaelz/simple-node-js-react-npm-app.git']])
            }
        }
        stage('Build App'){
            agent {
                docker {
                    image 'node:16'
                    args '-u root'
                    // Run the container on the node specified at the
                    // top-level of the Pipeline, in the same workspace,
                    // rather than on a new node entirely:
                    reuseNode true
                }
            }
            steps {
                echo "Building React App ..."
                sh 'npm install && npm run build'
            }
        }
        stage('Test') {
            agent {
                docker {
                    image 'node:16'
                    args '-u root'
                    // Run the container on the node specified at the
                    // top-level of the Pipeline, in the same workspace,
                    // rather than on a new node entirely:
                    reuseNode true
                }
            }
            steps {
                sh './jenkins/scripts/test.sh'
            }
        }
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ${NEXUS_URL}/${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }
        stage('Push Image to Nexus') {
            steps {
                echo 'Pushing Docker image to Nexus...'
                withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIALS_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        echo ${NEXUS_PASS} | docker login ${NEXUS_URL} -u ${NEXUS_USER} --password-stdin
                        docker push ${NEXUS_URL}/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        } 
        stage('Deploy to Docker Swarm') {
            steps {
                echo 'Deploying to Docker Swarm...'
                withCredentials([sshUserPrivateKey(credentialsId: "${SSH_CREDENTIALS_ID}", keyFileVariable: 'PK')]) {
                    sh '''
                        scp -i $PK -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" docker-swarm-compose.yml ${DOCKER_SWARM_MANAGER}:/tmp/${IMAGE_NAME}-docker-swarm-compose.yml
                        ssh -i $PK -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" ${DOCKER_SWARM_MANAGER} "
                        mkdir -pv /opt/ceph-data/SwarmDEV/mz-react-todo/
                        mv -v /tmp/${IMAGE_NAME}-docker-swarm-compose.yml /opt/ceph-data/SwarmDEV/mz-react-todo/docker-swarm-compose.yml
                        docker stack deploy mz-react-todo --compose-file /opt/ceph-data/SwarmDEV/mz-react-todo/docker-swarm-compose.yml
                        "
                    '''
                }
            }
        }
    }
    post {
        failure {
            echo 'This will run only if failed'
            mail to: 'michael.zhang@mclarens.co.nz',
                 subject: "Failed pipeline: ${currentBuild.fullDisplayName}",
                 body: "Something is wrong with ${env.BUILD_URL}"
        }
        cleanup {
            echo "Clean the workspace"
            deleteDir()
        }
    }
}
