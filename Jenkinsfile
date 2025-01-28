pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                withCredentials([
                    string(credentialsId: 'GIT_REPO_URL', variable: 'GIT_REPO_URL')
                ]) {
                    git branch: 'main',
                        url: "${env.GIT_REPO_URL}",
                        credentialsId: 'gitCreds'
                }
            }
        }

        stage('Add SSH Host Key') {
            steps {
                withCredentials([
                    string(credentialsId: 'REMOTE_HOST_IP', variable: 'REMOTE_HOST_IP')
                ]) {
                    sshagent(['mySSHKey']) {
                        sh """
                            mkdir -p ~/.ssh
                            ssh-keyscan -H ${env.REMOTE_HOST_IP} >> ~/.ssh/known_hosts
                        """
                    }
                }
            }
        }

        stage('Build & Push & Run (on remote)') {
            steps {
                sshagent(['mySSHKey']) {
                    withCredentials([
                        string(credentialsId: 'DOCKER_REGISTRY_URL', variable: 'DOCKER_REGISTRY_URL'),
                        string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE_NAME'),
                        string(credentialsId: 'APP_DIR', variable: 'APP_DIR'),
                        string(credentialsId: 'CONTAINER_NAME', variable: 'CONTAINER_NAME'),
                        string(credentialsId: 'HOST_PORT', variable: 'HOST_PORT'),
                        string(credentialsId: 'CONTAINER_PORT', variable: 'CONTAINER_PORT'),
                        usernamePassword(credentialsId: 'dockerCreds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS'),
                        string(credentialsId: 'sudoPass', variable: 'SUDO_PASS')
                    ]) {
                        ansiblePlaybook(
                            playbook: "deploy.yml",
                            inventory: "hosts",
                            colorized: true,
                            extraVars: [
                                registry_url: "${env.DOCKER_REGISTRY_URL}",
                                image_name:   "${env.DOCKER_IMAGE_NAME}",
                                docker_user:  "${env.DOCKER_USER}",
                                docker_pass:  "${env.DOCKER_PASS}",
                                sudoPass:     "${env.SUDO_PASS}",
                                
                                app_dir:      "${env.APP_DIR}",
                                container_name:"${env.CONTAINER_NAME}",
                                host_port:    "${env.HOST_PORT}",
                                container_port:"${env.CONTAINER_PORT}"
                            ]
                        )
                    }
                }
            }
        }

        stage('Test container (local curl)') {
            steps {
                withCredentials([
                    string(credentialsId: 'REMOTE_HOST_IP', variable: 'REMOTE_HOST_IP'),
                    string(credentialsId: 'HOST_PORT', variable: 'HOST_PORT')
                ]) {
                    script {
                        def response = sh(
                            script: """
                              curl -s -o /dev/null -w "%{http_code}" http://${env.REMOTE_HOST_IP}:${env.HOST_PORT}
                            """,
                            returnStdout: true
                        ).trim()

                        if (response != '200') {
                            error "Received HTTP status code ${response}, expected 200!"
                        }
                        echo "Container on ${env.REMOTE_HOST_IP}:${env.HOST_PORT} responded with HTTP 200, success!"
                    }
                }
            }
        }
    }
}
