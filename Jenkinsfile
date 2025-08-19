@Library('iti-sharedlib')_

pipeline {
    agent any
    
    options {
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_USER = credentials('docker-username')
        DOCKER_PASS = credentials('docker-password')
    }

    stages {
        stage("Get code") {
            steps {
                checkout scmGit(
                    branches: [[name: '*/master']], 
                    extensions: [], 
                    userRemoteConfigs: [[url: 'https://github.com/MahmoudEhab1/java.git']]
                )
            }
        }
        
        stage("Build app") {
            steps {
                script {
                    def mavenBuild = new org.iti.mvn()
                    mavenBuild.javaBuild("package install")
                }
            }
        }
        
        stage("Archive app") {
            steps {
                archiveArtifacts artifacts: '/*.jar', followSymlinks: false
            }
        }
        
        stage("Docker build") {
            steps {
                script {
                    def docker = new com.iti.docker()
                    docker.build("${env.DOCKER_IMAGE}", "${BUILD_NUMBER}")
                }
            }
        }
        
        stage("Push java app image") {
            steps {
                script {
                    def docker = new com.iti.docker()
                    sh "echo ${env.DOCKER_PASS} | docker login -u ${env.DOCKER_USER} --password-stdin"

                    docker.push("${env.DOCKER_IMAGE}", "${BUILD_NUMBER}")
                }
            }
        }
        
       stage("Update image") {
           steps {
                script {
                dir('argocd') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/main']],
                        extensions: [],
                        userRemoteConfigs: [[
                            url: 'https://github.com/MahmoudEhab1/argocd.git',
                            credentialsId: 'GITHUB-CREDENTIALS' 
                        ]]
                    ])
                    sh "sed -i 's#        image: .*#        image: ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER}#' iti-dev/deployment.yaml"
                    sh "git add ."
                    sh "git commit -m 'update Image'"
                    withCredentials([usernamePassword(
                        credentialsId: 'GITHUB-CREDENTIALS',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                )]) {
                        sh """
                            git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/MahmoudEhab1/argocd.git
                            git push origin HEAD:main
                            """
                }
                }
            }
    }
}
    }
}
