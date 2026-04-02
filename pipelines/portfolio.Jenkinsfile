pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
                    volumeMounts:
                    - name: docker-storage
                      mountPath: /var/lib/docker
                    - name: shared
                      mountPath: /shared
                  - name: kubectl
                    image: bitnami/kubectl:latest
                    command: ['sleep']
                    args: ['infinity']
                  - name: gcloud
                    image: google/cloud-sdk:slim
                    command: ['sleep']
                    args: ['infinity']
                    volumeMounts:
                    - name: shared
                      mountPath: /shared
                  volumes:
                  - name: docker-storage
                    emptyDir: {}
                  - name: shared
                    emptyDir: {}
            '''
        }
    }

    environment {
        PROJECT_ID     = 'project-e4ad9f18-82a4-4e98-ae4'
        REGION         = 'europe-west1'
        REGISTRY       = "${REGION}-docker.pkg.dev/${PROJECT_ID}/portfolio"
        IMAGE_NAME     = 'portfolio'
        IMAGE_TAG      = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'latest'}"
        FULL_IMAGE     = "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        LATEST_IMAGE   = "${REGISTRY}/${IMAGE_NAME}:latest"
        K8S_NAMESPACE  = 'portfolio'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                container('docker') {
                    sh "docker build -t ${FULL_IMAGE} -t ${LATEST_IMAGE} ."
                }
            }
        }

        stage('Auth to Artifact Registry') {
            steps {
                container('gcloud') {
                    withCredentials([file(credentialsId: 'gcp-service-account', variable: 'GCP_KEY')]) {
                        sh """
                            gcloud auth activate-service-account --key-file=\$GCP_KEY
                            gcloud auth print-access-token > /shared/gcp-token
                        """
                    }
                }
            }
        }

        stage('Push to Artifact Registry') {
            steps {
                container('docker') {
                    sh """
                        cat /shared/gcp-token | docker login -u oauth2accesstoken --password-stdin https://${REGION}-docker.pkg.dev
                        docker push ${FULL_IMAGE}
                        docker push ${LATEST_IMAGE}
                    """
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl set image deployment/portfolio \
                            portfolio=${FULL_IMAGE} \
                            -n ${K8S_NAMESPACE}
                        kubectl rollout status deployment/portfolio \
                            -n ${K8S_NAMESPACE} --timeout=120s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${FULL_IMAGE} successfully"
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}
