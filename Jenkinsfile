pipeline {
    agent any

    environment {
        CLIENT_ID     = credentials('FABRIC_CLIENT_ID')
        CLIENT_SECRET = credentials('FABRIC_CLIENT_SECRET')
        TENANT_ID     = credentials('FABRIC_TENANT_ID')

        WORKSPACE_ID  = "40c8fdea-6f13-4617-a454-1f39fb3ce2a4"
        MODEL_NAME    = "m9053-model"
        MODEL_FOLDER  = "sm-m9053-dev-001.SemanticModel"
        CONNECTION_ID = "f3519ff7-493e-4a2a-9020-57ffc1fd9bf5"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Get Token') {
            steps {
                script {
                    def tokenResponse = sh(script: """
                        curl -s -X POST https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token \
                        -H "Content-Type: application/x-www-form-urlencoded" \
                        -d grant_type=client_credentials \
                        -d client_id=${CLIENT_ID} \
                        -d client_secret=${CLIENT_SECRET} \
                        -d scope=https://api.fabric.microsoft.com/.default
                    """, returnStdout: true)

                    env.TOKEN = readJSON(text: tokenResponse).access_token
                }
            }
        }

        stage('Prepare Payload') {
            steps {
                script {

                    def pbismBase64 = sh(
                        script: "base64 -w 0 ${MODEL_FOLDER}/definition.pbism",
                        returnStdout: true
                    ).trim()

                    def findOut = sh(
                        script: "find ${MODEL_FOLDER}/definition -name '*.tmdl'",
                        returnStdout: true
                    ).trim()

                    def tmdlFiles = findOut ? findOut.split("\n") : []

                    def parts = [[
                        path: "definition.pbism",
                        payload: pbismBase64,
                        payloadType: "InlineBase64"
                    ]]

                    tmdlFiles.each { filePath ->
                        def relativePath = filePath.substring(filePath.indexOf("definition/"))
                        def fileBase64 = sh(script: "base64 -w 0 ${filePath}", returnStdout: true).trim()

                        parts << [
                            path: relativePath,
                            payload: fileBase64,
                            payloadType: "InlineBase64"
                        ]
                    }

                    writeJSON file: 'model_payload.json',
                        json: [
                            displayName: MODEL_NAME,
                            type: "SemanticModel",
                            definition: [parts: parts]
                        ]
                }
            }
        }

        stage('Delete If Exists') {
            steps {
                script {
                    def itemsResponse = sh(script: """
                        curl -s -X GET \
                        https://api.fabric.microsoft.com/v1/workspaces/${WORKSPACE_ID}/items \
                        -H "Authorization: Bearer ${env.TOKEN}"
                    """, returnStdout: true)

                    def itemsJson = readJSON text: itemsResponse

                    def existing = itemsJson.value.find {
                        it.displayName == MODEL_NAME && it.type == "SemanticModel"
                    }

                    if (existing) {
                        echo "Deleting existing model: ${existing.id}"
                        sh """
                            curl -s -X DELETE \
                            https://api.fabric.microsoft.com/v1/workspaces/${WORKSPACE_ID}/items/${existing.id} \
                            -H "Authorization: Bearer ${env.TOKEN}"
                        """
                    } else {
                        echo "No existing model found"
                    }
                }
            }
        }

        stage('Create Model') {
            steps {
                script {
                    def createResponse = sh(script: """
                        curl -s -X POST \
                        https://api.fabric.microsoft.com/v1/workspaces/${WORKSPACE_ID}/items \
                        -H "Authorization: Bearer ${env.TOKEN}" \
                        -H "Content-Type: application/json" \
                        -d @model_payload.json
                    """, returnStdout: true)

                    def createdJson = readJSON text: createResponse
                    env.SEMANTIC_MODEL_ID = createdJson.id

                    echo "Created Model ID: ${env.SEMANTIC_MODEL_ID}"
                }
            }
        }

        stage('Take Over') {
            steps {
                sh """
                    curl -s -X POST \
                    https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${env.SEMANTIC_MODEL_ID}/Default.TakeOver \
                    -H "Authorization: Bearer ${env.TOKEN}" \
                    -H "Content-Length: 0"
                """
            }
        }

        stage('Bind Connection') {
            steps {
                writeFile file: 'ds_payload.json', text: """
{
  "updateDetails": [
    {
      "datasourceSelector": {
        "datasourceType": "Extension"
      },
      "connectionId": "${CONNECTION_ID}"
    }
  ]
}
"""
                sh """
                    curl -s -X POST \
                    https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${env.SEMANTIC_MODEL_ID}/Default.UpdateDatasources \
                    -H "Authorization: Bearer ${env.TOKEN}" \
                    -H "Content-Type: application/json" \
                    -d @ds_payload.json
                """
            }
        }

        stage('Refresh') {
            steps {
                sh """
                    curl -s -X POST \
                    https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${env.SEMANTIC_MODEL_ID}/refreshes \
                    -H "Authorization: Bearer ${env.TOKEN}" \
                    -H "Content-Type: application/json" \
                    -d '{"type":"Full"}'
                """
            }
        }
    }

    post {
        always {
            sh "rm -f model_payload.json ds_payload.json"
        }
    }
}
