pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "praful2018/train-schedule"
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
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                script {
                    try {
                        kubernetesDeploy(
                            kubeconfigId: 'kubeconfig',
                            configs: 'train-schedule-kube-canary.yml',
                            enableConfigSubstitution: true
                        )
                    } catch (Exception deployException) {
                        echo "Canary Deployment Failed: ${deployException.message}"
                        // Handle the exception as needed
                        currentBuild.result = 'FAILURE'
                        error("Canary Deployment Failed")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                script {
                    try {
                        input 'Deploy to Production?'
                        milestone(1)
                        kubernetesDeploy(
                            kubeconfigId: 'kubeconfig',
                            configs: 'train-schedule-kube-canary.yml',
                            enableConfigSubstitution: true
                        )
                        kubernetesDeploy(
                            kubeconfigId: 'kubeconfig',
                            configs: 'train-schedule-kube.yml',
                            enableConfigSubstitution: true
                        )
                    } catch (Exception deployException) {
                        echo "Production Deployment Failed: ${deployException.message}"
                        // Handle the exception as needed
                        currentBuild.result = 'FAILURE'
                        error("Production Deployment Failed")
                    }
                }
            }
        }
    }
}
