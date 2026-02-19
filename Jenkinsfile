pipeline {
    agent any

    parameters {
        string(name: 'WORKSPACE_NAME', defaultValue: 'ws-m9053-uat-001', description: 'Target Fabric Workspace')
    }

    environment {
        FABRIC_CLIENT_ID     = credentials('FABRIC_CLIENT_ID')
        FABRIC_CLIENT_SECRET = credentials('FABRIC_CLIENT_SECRET')
        FABRIC_TENANT_ID     = credentials('FABRIC_TENANT_ID')
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Login to Fabric') {
            steps {
                sh '''
                    echo "Generating Fabric access token..."

                    ACCESS_TOKEN=$(curl -s -X POST https://login.microsoftonline.com/$FABRIC_TENANT_ID/oauth2/v2.0/token \
                      -H "Content-Type: application/x-www-form-urlencoded" \
                      -d "grant_type=client_credentials" \
                      -d "client_id=$FABRIC_CLIENT_ID" \
                      -d "client_secret=$FABRIC_CLIENT_SECRET" \
                      -d "scope=https://analysis.windows.net/powerbi/api/.default" | jq -r .access_token)

                    if [ "$ACCESS_TOKEN" = "null" ] || [ -z "$ACCESS_TOKEN" ]; then
                        echo "Failed to get access token"
                        exit 1
                    fi

                    echo $ACCESS_TOKEN > token.txt
                '''
            }
        }
        stage('Get Workspace ID') {
            steps {
                sh '''
                    TOKEN=$(cat token.txt)
        
                    echo "Looking for workspace: ${WORKSPACE_NAME}"
        
                    RESPONSE=$(curl -s -X GET \
                      -H "Authorization: Bearer $TOKEN" \
                      https://api.powerbi.com/v1.0/myorg/groups)
        
                    WORKSPACE_ID=$(echo "$RESPONSE" | jq -r --arg NAME "${WORKSPACE_NAME}" '.value[] | select(.name==$NAME) | .id')
        
                    if [ -z "$WORKSPACE_ID" ]; then
                        echo "Workspace not found!"
                        exit 1
                    fi
        
                    echo $WORKSPACE_ID > workspace_id.txt
                    echo "Workspace ID: $WORKSPACE_ID"
                '''
            }
        }


        
        stage('Validation Complete') {
            steps {
                sh '''
                    WORKSPACE_ID=$(cat workspace_id.txt)
                    echo "UAT Workspace Binding Successful: $WORKSPACE_ID"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline Succeeded"
        }
        failure {
            echo "Pipeline Failed"
        }
    }
}
