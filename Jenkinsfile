pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Choose the environment'
        )

        choice(
            name: 'CONFIG_TYPE',
            choices: ['node', 'both'],
            description: 'Update node JSON; both also validates environment JSON'
        )

        string(
            name: 'INSTANCE_TYPE',
            defaultValue: '',
            description: 'Optional: instance type; blank keeps current value'
        )

        string(
            name: 'K8S_VERSION',
            defaultValue: '',
            description: 'Optional: Kubernetes version; blank keeps current value'
        )

        string(
            name: 'CPU',
            defaultValue: '',
            description: 'Optional: CPU; blank keeps current value'
        )

        string(
            name: 'MEMORY',
            defaultValue: '',
            description: 'Optional: memory; blank keeps current value'
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
                    echo "Current branch:"
                    git branch --show-current
                    echo "Latest commit:"
                    git log -1 --oneline
                '''
            }
        }

        stage('Validate Configuration Files') {
            steps {
                script {
                    def nodeFile = "node/${params.ENVIRONMENT}.json"
                    def envFile = "environment/${params.ENVIRONMENT}.json"

                    if (!fileExists(nodeFile)) {
                        error("Node JSON file not found: ${nodeFile}")
                    }

                    readJSON file: nodeFile
                    echo "Node JSON is valid: ${nodeFile}"

                    if (params.CONFIG_TYPE == 'both') {
                        if (!fileExists(envFile)) {
                            error("Environment JSON file not found: ${envFile}")
                        }

                        readJSON file: envFile
                        echo "Environment JSON is valid: ${envFile}"
                    }
                }
            }
        }

        stage('Update Node Configuration') {
            steps {
                script {
                    def nodeFile = "node/${params.ENVIRONMENT}.json"
                    def nodeData = readJSON file: nodeFile
                    boolean updated = false

                    if (!(nodeData instanceof Map)) {
                        error("Expected a JSON object in ${nodeFile}")
                    }

                    if (params.INSTANCE_TYPE?.trim()) {
                        nodeData.instanceType = params.INSTANCE_TYPE.trim()
                        updated = true
                    }

                    if (params.K8S_VERSION?.trim()) {
                        if (!(nodeData.kubernetes instanceof Map)) {
                            error("Missing kubernetes object in ${nodeFile}")
                        }

                        nodeData.kubernetes.version = params.K8S_VERSION.trim()
                        updated = true
                    }

                    if (params.CPU?.trim()) {
                        if (!(nodeData.resources instanceof Map)) {
                            error("Missing resources object in ${nodeFile}")
                        }

                        nodeData.resources.cpu = params.CPU.trim()
                        updated = true
                    }

                    if (params.MEMORY?.trim()) {
                        if (!(nodeData.resources instanceof Map)) {
                            error("Missing resources object in ${nodeFile}")
                        }

                        nodeData.resources.memory = params.MEMORY.trim()
                        updated = true
                    }

                    if (!updated) {
                        error('All inputs are blank. Enter at least one value to update.')
                    }

                    writeJSON file: nodeFile,
                        json: nodeData,
                        pretty: 4

                    echo "Updated file: ${nodeFile}"
                }
            }
        }

        stage('Verify Updated JSON') {
            steps {
                script {
                    def nodeFile = "node/${params.ENVIRONMENT}.json"
                    def nodeData = readJSON file: nodeFile

                    echo "Updated node configuration:"
                    echo groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson(nodeData)
                    )

                    if (params.CONFIG_TYPE == 'both') {
                        def envFile = "environment/${params.ENVIRONMENT}.json"
                        readJSON file: envFile
                        echo "Environment JSON validation passed."
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

                    git add -- "node/${ENVIRONMENT}.json"

                    if git diff --cached --quiet; then
                        echo "No changes detected. The entered values may already exist."
                        exit 1
                    fi

                    git commit -m "Update ${ENVIRONMENT} node configuration"
                '''
            }
        }

        stage('Push Feature Branch') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Git-Flipkart',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        ASKPASS="$(mktemp)"
                        trap 'rm -f "$ASKPASS"' EXIT

                        cat > "$ASKPASS" <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) printf '%s\\n' "$GITHUB_USER" ;;
    *Password*) printf '%s\\n' "$GITHUB_TOKEN" ;;
esac
EOF

                        chmod 700 "$ASKPASS"

                        GIT_ASKPASS="$ASKPASS" \
                        GIT_TERMINAL_PROMPT=0 \
                        git push origin "$FEATURE_BRANCH"
                    '''
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
                        set -eu
                        set +x

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
    "title": f"Update {os.environ['ENVIRONMENT']} configuration",
    "head": branch,
    "base": base,
    "body": "Configuration update created by Jenkins."
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
        print("Pull request created successfully.")
        print("URL:", result["html_url"])
except urllib.error.HTTPError as error:
    details = error.read().decode("utf-8")
    raise SystemExit(
        f"GitHub API returned {error.code}: {details}"
    )
PY
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Changes pushed and pull request created.'
        }

        failure {
            echo 'FAILED: Check Console Output for the first error.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
