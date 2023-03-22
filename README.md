# Using Jenkins and Helm Chart to deploy Spring Boot Applications into EKS

### AWS Infrastructure

![Infrastructure](.files/EKS%20Veeva-AWS%20Infra.drawio.png)

### Jenkins Pipeline

![Infrastructure](.files/EKS%20Veeva-Jenkins.drawio.png)

Link: http://ec2-3-239-79-165.compute-1.amazonaws.com:8080/

# Used plugins

- Docker
- Docker Pipeline
- Kubernetes
- Kubernetes Credentials
- Kubernetes Client API
- GitHub Integration

# Pipeline scripts

NOTE: Update the AWS account ID.

CI
```groovy
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        registry = "012345678901.dkr.ecr.us-east-1.amazonaws.com/REPOSITORY_NAME"
        GIT_COMMIT = (sh(script: "git rev-parse HEAD", returnStdout: true)).trim()
    }

    stages {
        stage('Checkout the Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/ricardorqr/springboot-app']]])
            }
        }

        stage('Build the JAR') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage('Delete Old Images') {
            steps {
                script {
                    sh 'docker system prune'
                }
            }
        }

        stage('Building the Image') {
            steps {
                script {
                    dockerImage = docker.build registry
                }
            }
        }

        stage('Push into the ECR') {
            steps{
                script {
                    def commitId = GIT_COMMIT[0..5]
                    def buildNumber = "${env.BUILD_NUMBER}"

                    sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 012345678901.dkr.ecr.us-east-1.amazonaws.com'
                    sh "docker tag 012345678901.dkr.ecr.us-east-1.amazonaws.com/REPOSITORY_NAME 012345678901.dkr.ecr.us-east-1.amazonaws.com/REPOSITORY_NAME:${buildNumber}.${commitId}"
                    sh "docker push 012345678901.dkr.ecr.us-east-1.amazonaws.com/REPOSITORY_NAME:${buildNumber}.${commitId}"
                }
            }
        }
    }
}
```

CD
```groovy
pipeline {
    agent any

    environment {
        registry = "012345678901.dkr.ecr.us-east-1.amazonaws.com/REPOSITORY_NAME"
        GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    }

    stages {
        stage('Checkout the Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/ricardorqr/springboot-app']]])
            }
        }

        stage('Deploy into EKS') {
            steps{
                script {
                    kubeconfig(credentialsId: 'Kubeconfig', serverUrl: '') {
                        script {
                            def image = sh(returnStdout: true, script: """
                                aws ecr describe-images --repository-name REPOSITORY_NAME --region us-east-1 --query 'sort_by(imageDetails,& imagePushedAt)[-1].imageTags'
                            """).trim().substring(2)

                            sh("helm list -a")

                            def exists = sh(script: "kubectl get deployment springboot-app-helm", returnStatus: true) == 0

                            if (exists) {
                                echo "Update deployment image"
                                sh ("helm upgrade springboot-app helm --set=image.repository=012345678901.dkr.ecr.us-east-1.amazonaws.com/REPOSITORY_NAME:${image}")
                            } else {
                                echo "Create deployment and service manifest"
                                sh ('helm install springboot-app helm')
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### Kubernetes Infrastructure

Command to create EKS cluster.

```shell
eksctl create cluster \
	--profile devwithrico \
	--name test-eks \
	--region us-east-1 \
	--version 1.25 \
	--vpc-public-subnets subnet-03d19c83b26ead933,subnet-05e0d706717e3f048 \
	--nodegroup-name test-eks-node-group \
	--node-type t2.small \
	--nodes 1 \
	--nodes-min 1 \
	--nodes-max 3 \
	--node-volume-type gp2 \
	--node-volume-size 30 \
	--ssh-access \
	--ssh-public-key devwithrico \
	--asg-access
```

![Infrastructure](.files/EKS%20Veeva-Kubernetes.drawio.png)