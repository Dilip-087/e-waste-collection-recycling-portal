pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['Development', 'Staging', 'Production'], description: 'Target runtime cluster environment')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag value mapping')
    }

    environment {
        DOCKER_HUB_USER = 'dilip087' 
        IMAGE_NAME      = 'e-waste-collection-recycling-portal'
        KUBE_CONFIG     = credentials('kubernet') 
    }

    stages {
        stage('Checkout Code') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Quality Assurance & Testing') {
            parallel {
                stage('HTML Lint Verification') {
                    steps {
                        echo "Executing validation audits on index.html layout rules..."
                        bat "powershell -Command \"if ((Select-String -Path index.html -Pattern '<html').Count -and (Select-String -Path index.html -Pattern '</html>').Count) { exit 0 } else { exit 1 }\""
                    }
                }
                stage('Data Integrity Assertions') {
                    steps {
                        echo "Verifying presence of at least 5 configured e-waste centers..."
                        bat "powershell -Command \"if ((Select-String -Path index.html -Pattern 'class=\\\"card\\\"').Count -ge 5) { exit 0 } else { exit 1 }\""
                    }
                }
            }
        }

        stage('Kubernetes Orchestration') {
            steps {
                // Intentionally inject the broken image tag to trigger the cluster recovery mechanism
                bat "powershell -Command \"(Get-Content deployment.yaml) -replace 'dilip087/e-waste-collection-recycling-portal:latest', '%DOCKER_HUB_USER%/%IMAGE_NAME%:faulty-error-tag' | Set-Content deployment.yaml\""
                
                bat """
                    set KUBECONFIG=%KUBE_CONFIG%
                    kubectl apply -f deployment.yaml
                    
                    echo Tracking rolling update status...
                    kubectl rollout status deployment/ewaste-portal-deployment --timeout=30s
                    if %ERRORLEVEL% NEQ 0 (
                        echo CRITICAL: Rollout failed! Initiating safety rollback mechanics...
                        kubectl rollout undo deployment/ewaste-portal-deployment
                        exit /b 0
                    )
                """
            }
        }
    }
}