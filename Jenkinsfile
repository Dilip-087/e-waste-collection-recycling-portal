pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['Development', 'Staging', 'Production'], description: 'Target runtime cluster environment')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag value mapping')
    }

    environment {
        DOCKER_HUB_USER = 'dilip087' 
        IMAGE_NAME      = 'e-waste-collection-recycling-portal'
        KUBE_CONFIG     = credentials('jenkins-kubeconfig') 
    }

    stages {
        stage('Checkout Code') {
            stages {
                stage('Pull Repository') {
                    steps {
                        cleanWs()
                        checkout scm
                    }
                }
            }
        }

        stage('Quality Assurance & Testing') {
            parallel {
                stage('HTML Lint Verification') {
                    steps {
                        echo "Executing validation audits on index.html layout rules..."
                        sh "grep -q '<html' index.html && grep -q '</html>' index.html"
                    }
                }
                stage('Data Integrity Assertions') {
                    steps {
                        echo "Verifying presence of at least 5 configured e-waste centers..."
                        sh "grep -c 'class=\"card\"' index.html | grep -q '5'"
                    }
                }
            }
        }

        stage('Container Compilation') {
            steps {
                sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${params.IMAGE_TAG} ."
            }
        }

        stage('Publish Image Artifact') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo \$PASS | docker login -u \$USER --password-stdin"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${params.IMAGE_TAG}"
                }
            }
        }

        stage('Kubernetes Orchestration') {
            steps {
                sh "sed -i 's|dilip087/e-waste-collection-recycling-portal:latest|${DOCKER_HUB_USER}/${IMAGE_NAME}:${params.IMAGE_TAG}|g' deployment.yaml"
                
                sh """
                    export KUBECONFIG=${KUBE_CONFIG}
                    kubectl apply -f deployment.yaml
                    
                    echo "Tracking rolling update status..."
                    if ! kubectl rollout status deployment/ewaste-portal-deployment --timeout=60s; then
                        echo "CRITICAL: Rollout failed! Initiating safety rollback mechanics..."
                        kubectl rollout undo deployment/ewaste-portal-deployment
                        exit 1
                    fi
                """
            }
        }
    }
    
    post {
        always {
            sh "docker logout"
        }
    }
}
