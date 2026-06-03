pipeline {
    agent {
        docker { image 'devops-agent:latest' }
    }

    environment {
        APELLIDO          = "Anay"
        ACR_NAME          = "acrglobalcicd"
        ACR_LOGIN_SERVER  = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME        = "my-nodejs-app-${APELLIDO}"
        RESOURCE_GROUP    = "rg-cicd-terraform-app-baraujo03"
        AKS_NAME          = "aks-dev-eastus"
    }

    stages {

        // ─── CI ───────────────────────────────────────────────────────────────

        stage('[CI] Instalar dependencias') {
            steps {
                sh 'npm install'
            }
        }

        stage('[CI] Ejecutar pruebas unitarias') {
            steps {
                sh 'npm run test:unit'
            }
        }

        stage('[CI] Ejecutar pruebas de integración') {
            steps {
                sh 'npm run test:integration'
            }
        }

        stage('[CI] Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-clientId',       variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret',   variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId',       variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                        az login --service-principal \
                            --username="$AZ_CLIENT_ID" \
                            --password="$AZ_CLIENT_SECRET" \
                            --tenant="$AZ_TENANT_ID"
                        az account set --subscription "$AZ_SUBSCRIPTION_ID"
                    '''
                }
            }
        }

        stage('[CI] AKS Credentials') {
            steps {
                sh '''
                    az aks get-credentials \
                        --resource-group $RESOURCE_GROUP \
                        --name $AKS_NAME \
                        --overwrite-existing
                '''
            }
        }

        stage('[CI] Generar ID corto del commit') {
            steps {
                script {
                    env.IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "IMAGE_TAG: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                    az acr login --name $ACR_NAME
                    docker build -t $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG .
                    docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        // ─── CD-DEV ──────────────────────────────────────────────────────────

        stage('[CD-DEV] Deploy a AKS') {
            steps {
                script {
                    env.ENV             = "dev"
                    env.API_PROVIDER_URL = "https://dev.api.com"
                }
                sh '''
                    envsubst < k8s.yml > k8s-dev.yml
                    az aks command invoke \
                        --resource-group $RESOURCE_GROUP \
                        --name $AKS_NAME \
                        --command "kubectl apply -f k8s-dev.yml" \
                        --file k8s-dev.yml
                '''
            }
        }

        stage('[CD-DEV] Imprimir IP del servicio') {
            steps {
                sh '''
                    SERVICE_NAME="my-nodejs-service-${APELLIDO}-dev"
                    LB_IP=""
                    MAX_RETRIES=10
                    RETRY_COUNT=0
                    while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                        LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath="{.status.loadBalancer.ingress[0].ip}" 2>/dev/null || true)
                        if [ -z "$LB_IP" ]; then
                            RETRY_COUNT=$((RETRY_COUNT+1))
                            echo "Intento $RETRY_COUNT/$MAX_RETRIES: esperando IP..."
                            sleep 10
                        fi
                    done
                    echo "IP DEV: ${LB_IP:-No asignada aún}"
                '''
            }
        }

        // ─── APROBACIÓN QA ───────────────────────────────────────────────────

        stage('Aprobación QA') {
            steps {
                input message: '¿Aprobar deploy a QA?', ok: 'Aprobar'
            }
        }

        // ─── CD-QA ───────────────────────────────────────────────────────────

        stage('[CD-QA] Deploy a AKS') {
            steps {
                script {
                    env.ENV              = "qa"
                    env.API_PROVIDER_URL = "https://qa.api.com"
                }
                sh '''
                    envsubst < k8s.yml > k8s-qa.yml
                    az aks command invoke \
                        --resource-group $RESOURCE_GROUP \
                        --name $AKS_NAME \
                        --command "kubectl apply -f k8s-qa.yml" \
                        --file k8s-qa.yml
                '''
            }
        }

        stage('[CD-QA] Imprimir IP del servicio') {
            steps {
                sh '''
                    SERVICE_NAME="my-nodejs-service-${APELLIDO}-qa"
                    LB_IP=""
                    MAX_RETRIES=10
                    RETRY_COUNT=0
                    while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                        LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath="{.status.loadBalancer.ingress[0].ip}" 2>/dev/null || true)
                        if [ -z "$LB_IP" ]; then
                            RETRY_COUNT=$((RETRY_COUNT+1))
                            echo "Intento $RETRY_COUNT/$MAX_RETRIES: esperando IP..."
                            sleep 10
                        fi
                    done
                    echo "IP QA: ${LB_IP:-No asignada aún}"
                '''
            }
        }

        // ─── APROBACIÓN PRD ──────────────────────────────────────────────────

        stage('Aprobación PRD') {
            steps {
                input message: '¿Aprobar deploy a PRD?', ok: 'Aprobar'
            }
        }

        // ─── CD-PRD ──────────────────────────────────────────────────────────

        stage('[CD-PRD] Deploy a AKS') {
            steps {
                script {
                    env.ENV              = "prd"
                    env.API_PROVIDER_URL = "https://prd.api.com"
                }
                sh '''
                    envsubst < k8s.yml > k8s-prd.yml
                    az aks command invoke \
                        --resource-group $RESOURCE_GROUP \
                        --name $AKS_NAME \
                        --command "kubectl apply -f k8s-prd.yml" \
                        --file k8s-prd.yml
                '''
            }
        }

        stage('[CD-PRD] Imprimir IP del servicio') {
            steps {
                sh '''
                    SERVICE_NAME="my-nodejs-service-${APELLIDO}-prd"
                    LB_IP=""
                    MAX_RETRIES=10
                    RETRY_COUNT=0
                    while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                        LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath="{.status.loadBalancer.ingress[0].ip}" 2>/dev/null || true)
                        if [ -z "$LB_IP" ]; then
                            RETRY_COUNT=$((RETRY_COUNT+1))
                            echo "Intento $RETRY_COUNT/$MAX_RETRIES: esperando IP..."
                            sleep 10
                        fi
                    done
                    echo "IP PRD: ${LB_IP:-No asignada aún}"
                '''
            }
        }
    }
}
