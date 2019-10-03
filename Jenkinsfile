pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "pio2pio/train-schedule"
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when { branch 'master' }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login_jenkinsid') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when        { branch 'master'     }
            environment { CANARY_REPLICAS = 2 }
            steps {
                input 'Deploy to Canary?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig_jenkinsId',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
        }
        stage('DeployToProduction') {
            when { branch 'master' }
            environment { CANARY_REPLICAS = 0 }
            steps {
                input 'Deploy to Production?'
                milestone(2)

                // down-scaling canary pods by passing 'CANARY_REPLICAS = 0'
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig_jenkinsId',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
                
                // deploy production pods and service
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig_jenkinsId',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}
