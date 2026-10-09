pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Select the environment to update'
        )

        choice(
            name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Select which configuration to process'
        )

        string(
            name: 'INSTANCE_TYPE',
            defaultValue: 't3.medium',
            description: 'EC2 instance type'
        )

        string(
            name: 'K8S_VERSION',
            defaultValue: '1.30',
            description: 'Kubernetes version'
        )

        string(
            name: 'CPU',
            defaultValue: '500m',
            description: 'CPU resource value'
        )

        string(
            name: 'MEMORY',
            defaultValue: '512Mi',
            description: 'Memory resource value'
        )
    }

    environment {
        GIT_REPO = 'https://github.com/Vineeth7861/flipkart.git'
        GITHUB_REPO = 'Vineeth7861/flipkart'
        BASE_BRANCH = 'main'
        GIT_CREDENTIALS_ID = 'Git-Flipkart'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BASE_BRANCH}",
                    credentialsId: "${GIT_CREDENTIALS_ID}",
                    url: "${GIT_REPO}"

                sh '''
                    git branch --show-current
                    git log -1 --oneline
                '''
            }
        }

        stage('Validate Configuration Files') {
            steps {
                script {
                    def envFile = "environment/${params.ENVIRONMENT}.json"
                    def nodeFile = "node/${params.ENVIRONMENT}.json"

                    if (params.CONFIG_TYPE in ['both', 'environment']) {
                        if (!fileExists(envFile)) {
                            error("Environment file not found: ${envFile}")
                        }

                        readJSON file: envFile
                        echo "Environment JSON is valid: ${envFile}"
                    }

                    if (params.CONFIG_TYPE in ['both', 'node']) {
                        if (!fileExists(nodeFile)) {
                            error("Node file not found: ${nodeFile}")
                        }

                        readJSON file: nodeFile
                        echo "Node JSON is valid: ${nodeFile}"
                    }
                }
            }
        }

        stage('Update Node Configuration') {
            when {
                expression {
                    params.CONFIG_TYPE in ['both', 'node']
                }
            }

            steps {
                script {
                    def nodeFile = "node/${params.ENVIRONMENT}.json"
                    def nodeData = readJSON file: nodeFile

                    nodeData.instanceType = params.INSTANCE_TYPE

                    if (nodeData.kubernetes instanceof Map) {
                        nodeData.kubernetes.version = params.K8S_VERSION
                    } else {
                        error("Expected a 'kubernetes' object in ${nodeFile}")
                    }

                    if (nodeData.resources instanceof Map) {
                        nodeData.resources.cpu = params.CPU
                        nodeData.resources.memory = params.MEMORY
                    } else {
                        error("Expected a 'resources' object in ${nodeFile}")
                    }

                    writeJSON file: nodeFile,
                        json: nodeData,
                        pretty: 4

                    echo "Updated node configuration: ${nodeFile}"
                }
            }
        }

        stage('Verify Configuration') {
            steps {
                script {
                    if (params.CONFIG_TYPE in ['both', 'environment']) {
                        def envFile = "environment/${params.ENVIRONMENT}.json"
                        readJSON file: envFile
                        echo "Verified: ${envFile}"
                    }

                    if (params.CONFIG_TYPE in ['both', 'node']) {
                        def nodeFile = "node/${params.ENVIRONMENT}.json"
                        def nodeData = readJSON file: nodeFile

                        echo "Instance type: ${nodeData.instanceType}"
                        echo "Kubernetes version: ${nodeData.kubernetes.version}"
                        echo "CPU: ${nodeData.resources.cpu}"
                        echo "Memory: ${nodeData.resources.memory}"
                    }
                }
            }
        }

        stage('Create Feature Branch and Commit') {
            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"
                }

                sh '''
                    set -eu

                    git config user.name "Vineeth Jhonny"
                    git config user.email "vineeth19@gmail.com"

                    git switch -c "$FEATURE_BRANCH"

                    git add environment/*.json node/*.json

                    if git diff --cached --quiet; then
                        echo "No configuration changes to commit."
                        exit 1
                    fi

                    git commit -m "Update ${ENVIRONMENT} configuration"
                '''
            }
        }

        stage('Push Feature Branch') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: env.GIT_CREDENTIALS_ID,
                            usernameVariable: 'GITHUB_USER',
                            passwordVariable: 'GITHUB_TOKEN'
                        )
                    ]) {
                        sh '''
                            set -eu

                            git -c credential.helper= \
                                -c credential.helper='!f() { echo username=$GITHUB_USER; echo password=$GITHUB_TOKEN; }; f' \
                                push origin "$FEATURE_BRANCH"
                        '''
                    }
                }
            }
        }

        stage('Create GitHub Pull Request') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Git-Flipkart',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        python3 - <<'PY'
import json
import os
import urllib.request
import urllib.error

repo = "Vineeth7861/flipkart"
branch = os.environ["FEATURE_BRANCH"]
base = os.environ["BASE_BRANCH"]
token = os.environ["GITHUB_TOKEN"]

payload = {
    "title": f"Update configuration: {branch}",
    "head": branch,
    "base": base,
    "body": "Automated configuration update by Jenkins."
}

request = urllib.request.Request(
    f"https://api.github.com/repos/{repo}/pulls",
    data=json.dumps(payload).encode("utf-8"),
    headers={
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28"
    },
    method="POST"
)

try:
    with urllib.request.urlopen(request) as response:
        result = json.loads(response.read().decode("utf-8"))
        print("Pull request created:", result["html_url"])
except urllib.error.HTTPError as error:
    details = error.read().decode("utf-8")
    raise SystemExit(
        f"GitHub API error {error.code}: {details}"
    )
PY
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            echo 'Check your GitHub repository for the pull request.'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
