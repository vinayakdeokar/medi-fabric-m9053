pipeline {
    agent any

    parameters {
        string(name: 'WORKSPACE_NAME', defaultValue: 'ws-m9053-uat-001', description: 'Target Fabric Workspace')
    }

    environment {
        FABRIC_CLIENT_ID     = credentials('fabric-client-id')
        FABRIC_CLIENT_SECRET = credentials('fabric-client-secret')
        FABRIC_TENANT_ID     = credentials('fabric-tenant-id')
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "Checking out repository..."
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    if ! command -v jq &> /dev/null
                    then
                        echo "Installing jq..."
                        apt-get update && apt-get install -y jq
                    fi
                '''
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

                    echo "Fetching workspace ID for: ${WORKSPACE_NAME}"

                    WORKSPACE_ID=$(curl -s -X GET \
                      -H "Authorization: Bearer $TOKEN" \
                      https://api.powerbi.com/v1.0/myorg/groups | \
                      jq -r ".value[] | select(.name==\"${WORKSPACE_NAME}\") | .id")

                    if [ -z "$WORKSPACE_ID" ]; then
                        echo "Workspace not found!"
                        exit 1
                    fi

                    echo $WORKSPACE_ID > workspace_id.txt
                    echo "Workspace ID: $WORKSPACE_ID"
                '''
            }
        }

        stage('UAT Binding Validation') {
            steps {
                sh '''
                    WORKSPACE_ID=$(cat workspace_id.txt)
                    echo "Ready to deploy to Workspace ID: $WORKSPACE_ID"
                '''
            }
        }
    }

    post {
        success {
            echo "UAT Binding Validation Successful"
        }
        failure {
            echo "Pipeline Failed"
        }
    }
}
