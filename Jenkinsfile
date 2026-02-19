pipeline {
    agent any

    environment {
        FABRIC_CLIENT_ID     = credentials('fabric-client-id')
        FABRIC_CLIENT_SECRET = credentials('fabric-client-secret')
        FABRIC_TENANT_ID     = credentials('fabric-tenant-id')

        WORKSPACE_NAME = "ws-m9053-uat-001"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "Checking out source from Git..."
                checkout scm
            }
        }

        stage('Login to Fabric') {
            steps {
                sh '''
                    echo "Logging into Fabric..."

                    ACCESS_TOKEN=$(curl -s -X POST https://login.microsoftonline.com/$FABRIC_TENANT_ID/oauth2/v2.0/token \
                      -H "Content-Type: application/x-www-form-urlencoded" \
                      -d "grant_type=client_credentials" \
                      -d "client_id=$FABRIC_CLIENT_ID" \
                      -d "client_secret=$FABRIC_CLIENT_SECRET" \
                      -d "scope=https://analysis.windows.net/powerbi/api/.default" | jq -r .access_token)

                    echo $ACCESS_TOKEN > token.txt
                '''
            }
        }

        stage('Deploy to UAT Workspace') {
            steps {
                sh '''
                    echo "Deploying artifacts to $WORKSPACE_NAME"

                    TOKEN=$(cat token.txt)

                    # Example API call (placeholder)
                    curl -X GET \
                      -H "Authorization: Bearer $TOKEN" \
                      https://api.powerbi.com/v1.0/myorg/groups
                '''
            }
        }

        stage('Deployment Complete') {
            steps {
                echo "UAT Deployment Finished Successfully"
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed"
        }
        success {
            echo "Pipeline succeeded"
        }
    }
}
