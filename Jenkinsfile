pipeline {
    agent any
    tools {
        maven "MAVEN"
    }
    parameters {
        choice(
            choices: ['New Release', 'Use Previous Version'],
            description: 'Select the release option',
            name: 'RELEASE_OPTION'
        )
    }
    triggers {
        // Poll repository every 2 minutes for changes
        pollSCM('H/2 * * * *')
    }
    stages {
        stage('Git Clone ') {
            steps {
                script {
                    def releaseOption = params.RELEASE_OPTION
                    if (releaseOption == 'New Release') {
                        git branch: 'main', credentialsId: 'spring-Bjit-Github-SSHCredentials', url: 'git@github.com:nazmul-islam76/spring-bjit.git'
                    } else {
                        echo "Skipping Git Clone stage for Use Previous Version"
                    }
                }
            }
        }

        stage('Building Jar File from GIT') {
            steps {
                script {
                    def releaseOption = params.RELEASE_OPTION
                    if (releaseOption == 'New Release') {
                        sh 'mvn clean install'
                    } else {
                        echo "Skipping Building Jar File stage for Use Previous Version"
                    }
                }
            }
        }

        stage('Delete Previous Images from Local Docker') {
            steps {
                script {
                    def releaseOption = params.RELEASE_OPTION
                    if (releaseOption == 'New Release') {
                        def imageName = 'nazmulislam76/spring-bjit'
                        
                        // Get the IDs of previous images with the same name and tag 'latest'
                        def imageIds = sh(
                            script: "docker images --filter=reference='${imageName}:latest' --format '{{.ID}}'",
                            returnStdout: true
                        ).trim()
                        
                        // Delete previous images
                        if (imageIds) {
                            sh "docker rmi -f ${imageIds}"
                        } else {
                            echo "No previous images found for ${imageName}:latest"
                        }
                    } else {
                        echo "Skipping Deletion of previous version"
                    }
                }
            }
        }
        stage('Docker Jar Image Build') {
            steps {
                script {
                    def dockerfilePathend = "docker-build/Dockerfile"
                    def tag = ""
                    def releaseOption = params.RELEASE_OPTION

                    if (releaseOption == 'New Release') {
                        // Fetch the tags for the Docker image from Docker Hub API
                        def tagsJson = sh(
                            script: 'curl -s "https://hub.docker.com/v2/repositories/nazmulislam76/spring-bjit/tags/?page_size=100" | jq -r ".results[].name"',
                            returnStdout: true
                        ).trim()
                        def highestNumberedTag = 0
                        for (def t in tagsJson.split('\n')) {
                            if (t.isNumber() && t.toInteger() > highestNumberedTag) {
                                highestNumberedTag = t.toInteger()
                            }
                        }
                        tag = (highestNumberedTag + 1).toString()
                    } else if (releaseOption == 'Use Previous Version') {
                        echo "Skipping Building Push and Build File stage for Use Previous Version"
                    }

                    if (tag) {
                        def springEnd = docker.build("nazmulislam76/spring-bjit:${tag}", "-f ${dockerfilePathend} .")
                        withDockerRegistry(credentialsId: 'DockerHubCredential', url: '') {
                            springEnd.push(tag)
                            springEnd.push("latest")
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes Pods') {
            steps {
                script {
                    def releaseOption = params.RELEASE_OPTION
                    if (releaseOption == 'New Release') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s-credential', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            def sqlDeploymentExists = sh(
                                script: 'kubectl get deployment -l app=mysql -o name',
                                returnStatus: true
                            )
                            if (sqlDeploymentExists == 0) {
                                try {
                                    sh 'kubectl delete -f k8s-code/sql-deployment.yaml'
                                } catch (Exception e) {
                                    echo "Error deleting SQL deployment: ${e.getMessage()}"
                                }
                            }

                            def jarDeploymentExists = sh(
                                script: 'kubectl get deployment -l app=openjdk -o name',
                                returnStatus: true
                            )
                            if (jarDeploymentExists == 0) {
                                try {
                                    sh 'kubectl delete -f k8s-code/jar-deployment.yaml'
                                } catch (Exception e) {
                                    echo "Error deleting JAR deployment: ${e.getMessage()}"
                                }
                            }

                            sh 'kubectl apply -f k8s-code/sql-deployment.yaml'
                            sh 'kubectl apply -f k8s-code/jar-deployment.yaml'
                        }
                    } else {
                        echo "Skipping Deploy to Kubernetes stage for Use Previous Version"
                    }
                }
            }
        }
    }
}
