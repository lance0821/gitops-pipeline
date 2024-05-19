pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/label: jnlp-slave
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    tty: true
    volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
  - name: podman
    image: quay.io/podman/stable
    command:
      - cat
    tty: true
    securityContext:
      privileged: true
    volumeMounts:
      - name: podman-sock
        mountPath: /run/podman/podman.sock
      - name: podman-storage
        mountPath: /var/lib/containers/storage
  volumes:
  - name: workspace-volume
    emptyDir: {}
  - name: podman-sock
    hostPath:
      path: /run/podman/podman.sock
      type: Socket
  - name: podman-storage
    emptyDir: {}
"""
        }
    }

    environment {
        APP_NAME = 'devops-pipeline'
        DOCKER_USERNAME = 'lance0821'
        REPOSITORY_NAME = 'devops-pipeline'
        IMAGE_NAME = "docker.io/${DOCKER_USERNAME}/${REPOSITORY_NAME}"
    }

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'The image tag to deploy')
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code From Github') {
            steps {
                git branch: 'main', credentialsId: 'github-credentials', url: 'https://github.com/lance0821/gitops-pipeline.git'
            }
        }

        stage('Update the Deployment Tags') {
            steps {
                script {
                    def newTag = params.IMAGE_TAG
                    def fullImageName = "${IMAGE_NAME}:${newTag}"
                    def fileContent = readFile 'deployment.yaml'
                    
                    // Print the original content for debugging
                    println("Original deployment.yaml content:\n${fileContent}")
                    
                    // Correct replacement to avoid duplicating the repository path
                    def updatedContent = fileContent.replaceAll(/(image:.*\/.*):.*/, "\$1:${newTag}")
                    
                    // Print the updated content for debugging
                    println("Updated deployment.yaml content:\n${updatedContent}")
                    
                    writeFile file: 'deployment.yaml', text: updatedContent
                }
            }
        }

        stage('Push the changed deployment file to git') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GITHUB_PASSWORD', usernameVariable: 'GITHUB_USER')]) {
                    script {
                        def gitUrl = "https://${URLEncoder.encode(GITHUB_USER, 'UTF-8')}:${URLEncoder.encode(GITHUB_PASSWORD, 'UTF-8')}@github.com/lance0821/gitops-pipeline.git"
                        def commitMessage = "Updated deployment.yaml with new image tag: ${params.IMAGE_TAG}"
                        sh """
                        git config --global user.email "lance0821@gmail.com"
                        git config --global user.name "Lance Lewandowski"
                        git pull origin main
                        git add deployment.yaml
                        git commit -m "${commitMessage}"
                        git push ${gitUrl} main
                        """
                    }
                }
            }
        }
    }
}
