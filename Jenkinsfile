pipeline {
    agent any

    parameters {
        string(name: 'WORKSPACE_ID', description: 'Target Fabric Workspace ID')
        string(name: 'SEMANTIC_MODEL_ID', description: 'Semantic Model Item ID')
        string(name: 'MODEL_FOLDER', defaultValue: 'semantic-models/m9053.SemanticModel', description: 'Semantic Model Folder Path')
        string(name: 'CONNECTION_ID', description: 'Databricks Connection ID')
    }

    environment {
        CLIENT_ID     = credentials('FABRIC_CLIENT_ID')
        CLIENT_SECRET = credentials('FABRIC_CLIENT_SECRET')
        TENANT_ID     = credentials('FABRIC_TENANT_ID')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Get Fabric Token') {
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

        stage('Prepare Model Payload') {
            steps {
                script {

                    def pbismBase64 = sh(
                        script: "base64 -w 0 ${params.MODEL_FOLDER}/definition.pbism",
                        returnStdout: true
                    ).trim()

                    def parts = [
                        [
                            path: "definition.pbism",
                            payload: pbismBase64,
                            payloadType: "InlineBase64"
                        ]
                    ]

                    def tmdlFiles = sh(
                        script: "find ${params.MODEL_FOLDER}/definition -name '*.tmdl'",
                        returnStdout: true
                    ).trim().split("\n")

                    tmdlFiles.each { filePath ->

                        def relativePath = filePath.substring(
                            filePath.indexOf("definition/")
                        )

                        def fileBase64 = sh(
                            script: "base64 -w 0 ${filePath}",
                            returnStdout: true
                        ).trim()

                        parts << [
                            path: relativePath,
                            payload: fileBase64,
                            payloadType: "InlineBase64"
                        ]
                    }

                    writeJSON file: 'model_payload.json',
                        json: [
                            displayName: "m9053-model",
                            type: "SemanticModel",
                            definition: [parts: parts]
                        ]
                }
            }
        }

        stage('Deploy Semantic Model') {
            steps {
                sh """
                    curl -s -X POST \
                    https://api.fabric.microsoft.com/v1/workspaces/${params.WORKSPACE_ID}/items/${params.SEMANTIC_MODEL_ID}/updateDefinition \
                    -H "Authorization: Bearer ${env.TOKEN}" \
                    -H "Content-Type: application/json" \
                    -d @model_payload.json
                """
            }
        }

        stage('Take Over Dataset') {
            steps {
                sh """
                    curl -s -X POST \
                    https://api.powerbi.com/v1.0/myorg/groups/${params.WORKSPACE_ID}/datasets/${params.SEMANTIC_MODEL_ID}/Default.TakeOver \
                    -H "Authorization: Bearer ${env.TOKEN}" \
                    -H "Content-Length: 0"
                """
            }
        }

        stage('Bind Databricks Connection') {
            steps {
                script {

                    writeFile file: 'ds_payload.json', text: """
{
  "updateDetails": [
    {
      "datasourceSelector": {
        "datasourceType": "Extension",
        "connectionDetails": {
          "extensionDataSourceKind": "Databricks",
          "extensionDataSourcePath": {
            "host": "adb-7405618110977329.9.azuredatabricks.net",
            "httpPath": "/sql/1.0/warehouses/334a2ae248719051"
          }
        }
      },
      "connectionId": "${params.CONNECTION_ID}"
    }
  ]
}
"""

                    sh """
                        curl -s -X POST \
                        https://api.powerbi.com/v1.0/myorg/groups/${params.WORKSPACE_ID}/datasets/${params.SEMANTIC_MODEL_ID}/Default.UpdateDatasources \
                        -H "Authorization: Bearer ${env.TOKEN}" \
                        -H "Content-Type: application/json" \
                        -d @ds_payload.json
                    """
                }
            }
        }

        stage('Trigger Refresh') {
            steps {
                sh """
                    curl -s -X POST \
                    https://api.powerbi.com/v1.0/myorg/groups/${params.WORKSPACE_ID}/datasets/${params.SEMANTIC_MODEL_ID}/refreshes \
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
