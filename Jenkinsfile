pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Choose the environment configuration to update'
        )
        choice(
            name: 'CONFIG_TYPE',
            choices: ['node', 'environment', 'both'],
            description: 'Choose which configuration files to update'
        )

        string(name: 'INSTANCE_TYPE', defaultValue: '', description: 'Node only; blank keeps the existing value, for example t3.micro')
        string(name: 'K8S_VERSION', defaultValue: '', description: 'Node only; blank keeps the existing value, for example 1.30.0')
        string(name: 'CPU', defaultValue: '', description: 'Node only; blank keeps the existing value, for example 500m or 1')
        string(name: 'MEMORY', defaultValue: '', description: 'Node only; blank keeps the existing value, for example 512Mi or 1Gi')

        string(name: 'ENV_APPNAME', defaultValue: '', description: 'Environment only; blank keeps the existing value')
        string(name: 'ENV_VERSION', defaultValue: '', description: 'Environment only; blank keeps the existing value, for example 2.1.0')
        string(name: 'ENV_REPLICAS', defaultValue: '', description: 'Environment only; blank keeps the existing value, for example 2')
        choice(
            name: 'ENV_LOGLEVEL',
            choices: ['KEEP', 'DEBUG', 'INFO', 'WARN', 'ERROR'],
            description: 'Environment only; KEEP preserves the current value'
        )
    }

    environment {
        GIT_REPO = 'https://github.com/Vineeth7861/flipkart.git'
        GITHUB_REPO = 'Vineeth7861/flipkart'
        BASE_BRANCH = 'main'
        GIT_CREDENTIALS_ID = 'Git-Flipkart'
        GIT_COMMIT_NAME = 'Vineeth Jhonny'
        GIT_COMMIT_EMAIL = 'vineeth19@gmail.com'
    }

    stages {
        stage('Checkout latest main') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: env.GIT_REPO,
                        credentialsId: env.GIT_CREDENTIALS_ID
                    ]]
                ])

                sh '''
                    set -eu
                    git fetch origin main
                    git checkout -B main origin/main
                    git config user.name "$GIT_COMMIT_NAME"
                    git config user.email "$GIT_COMMIT_EMAIL"
                    echo "Checked out the latest main branch."
                    git rev-parse --short HEAD
                '''
            }
        }

        stage('Validate JSON and inputs') {
            steps {
                script {
                    env.SELECTED_ENV = params.ENVIRONMENT.trim()
                    env.SELECTED_CONFIG_TYPE = params.CONFIG_TYPE.trim()

                    def nodeFile = "node/${env.SELECTED_ENV}.json"
                    def environmentFile = "environment/${env.SELECTED_ENV}.json"

                    if (env.SELECTED_CONFIG_TYPE in ['node', 'both']) {
                        if (!fileExists(nodeFile)) {
                            error("Required file not found: ${nodeFile}")
                        }

                        def nodeData = readJSON(file: nodeFile)
                        if (!(nodeData instanceof Map)) {
                            error("${nodeFile} must contain a JSON object.")
                        }

                        if (params.INSTANCE_TYPE.trim() &&
                            !(params.INSTANCE_TYPE.trim() ==~ /^[a-z][a-z0-9]*\\.[a-z0-9]+$/)) {
                            error('INSTANCE_TYPE must look like t3.micro or m7i.large.')
                        }

                        if (params.K8S_VERSION.trim() &&
                            !(params.K8S_VERSION.trim() ==~ /^v?[0-9]+\\.[0-9]+\\.[0-9]+([+-][0-9A-Za-z.-]+)?$/)) {
                            error('K8S_VERSION must look like 1.30.0 or v1.30.0.')
                        }

                        if (params.CPU.trim() &&
                            !(params.CPU.trim() ==~ /^[0-9]+m$|^[0-9]+(\\.[0-9]+)?$/)) {
                            error('CPU must be a number such as 1 or a millicore value such as 500m.')
                        }

                        if (params.MEMORY.trim() &&
                            !(params.MEMORY.trim() ==~ /^[0-9]+(Ki|Mi|Gi|Ti|K|M|G|T)?$/)) {
                            error('MEMORY must be a value such as 512Mi, 1Gi, or 1024.')
                        }
                    }

                    if (env.SELECTED_CONFIG_TYPE in ['environment', 'both']) {
                        if (!fileExists(environmentFile)) {
                            error("Required file not found: ${environmentFile}")
                        }

                        def environmentData = readJSON(file: environmentFile)
                        if (!(environmentData instanceof Map)) {
                            error("${environmentFile} must contain a JSON object.")
                        }

                        if (params.ENV_APPNAME.trim() &&
                            !(params.ENV_APPNAME.trim() ==~ /^[A-Za-z0-9._-]+$/)) {
                            error('ENV_APPNAME can contain letters, numbers, dots, underscores, and hyphens.')
                        }

                        if (params.ENV_VERSION.trim() &&
                            !(params.ENV_VERSION.trim() ==~ /^[0-9]+\\.[0-9]+\\.[0-9]+([+-][0-9A-Za-z.-]+)?$/)) {
                            error('ENV_VERSION must look like 2.1.0.')
                        }

                        if (params.ENV_REPLICAS.trim()) {
                            if (!(params.ENV_REPLICAS.trim() ==~ /^[0-9]+$/) ||
                                params.ENV_REPLICAS.trim().toInteger() < 1) {
                                error('ENV_REPLICAS must be a whole number of at least 1.')
                            }
                        }
                    }

                    echo "Selected environment: ${env.SELECTED_ENV}"
                    echo "Selected configuration type: ${env.SELECTED_CONFIG_TYPE}"
                    echo 'Input and selected JSON file checks passed.'
                }
            }
        }

        stage('Create feature branch') {
            steps {
                script {
                    env.FEATURE_BRANCH = "feature/update-${env.SELECTED_ENV}-${env.BUILD_NUMBER}"
                    echo "Feature branch: ${env.FEATURE_BRANCH}"
                }

                sh '''
                    set -eu
                    git checkout -b "$FEATURE_BRANCH" origin/main
                '''
            }
        }

        stage('Update selected JSON files') {
            steps {
                script {
                    if (env.SELECTED_CONFIG_TYPE in ['node', 'both']) {
                        def nodeFile = "node/${env.SELECTED_ENV}.json"
                        def nodeData = readJSON(file: nodeFile)

                        if (params.INSTANCE_TYPE.trim()) {
                            nodeData.instanceType = params.INSTANCE_TYPE.trim()
                        }

                        if (params.K8S_VERSION.trim()) {
                            if (!(nodeData.kubernetes instanceof Map)) {
                                error("${nodeFile} must contain a kubernetes JSON object.")
                            }
                            nodeData.kubernetes.version = params.K8S_VERSION.trim()
                        }

                        if (params.CPU.trim()) {
                            if (!(nodeData.resources instanceof Map)) {
                                error("${nodeFile} must contain a resources JSON object.")
                            }
                            nodeData.resources.cpu = params.CPU.trim()
                        }

                        if (params.MEMORY.trim()) {
                            if (!(nodeData.resources instanceof Map)) {
                                error("${nodeFile} must contain a resources JSON object.")
                            }
                            nodeData.resources.memory = params.MEMORY.trim()
                        }

                        writeJSON(file: nodeFile, json: nodeData, pretty: 2)
                        echo "Updated selected values in ${nodeFile}; blank inputs were left unchanged."
                    }

                    if (env.SELECTED_CONFIG_TYPE in ['environment', 'both']) {
                        def environmentFile = "environment/${env.SELECTED_ENV}.json"
                        def environmentData = readJSON(file: environmentFile)

                        if (params.ENV_APPNAME.trim()) {
                            environmentData.appName = params.ENV_APPNAME.trim()
                        }

                        if (params.ENV_VERSION.trim()) {
                            environmentData.version = params.ENV_VERSION.trim()
                        }

                        if (params.ENV_REPLICAS.trim()) {
                            environmentData.replicas = params.ENV_REPLICAS.trim().toInteger()
                        }

                        if (params.ENV_LOGLEVEL != 'KEEP') {
                            environmentData.logLevel = params.ENV_LOGLEVEL
                        }

                        writeJSON(file: environmentFile, json: environmentData, pretty: 2)
                        echo "Updated selected values in ${environmentFile}; blank inputs and KEEP were left unchanged."
                    }
                }
            }
        }

        stage('Review changes') {
            steps {
                script {
                    def filesToReview = []

                    if (env.SELECTED_CONFIG_TYPE in ['node', 'both']) {
                        filesToReview.add("node/${env.SELECTED_ENV}.json")
                    }
                    if (env.SELECTED_CONFIG_TYPE in ['environment', 'both']) {
                        filesToReview.add("environment/${env.SELECTED_ENV}.json")
                    }

                    withEnv(["FILES_TO_REVIEW=${filesToReview.join(' ')}"]) {
                        sh '''
                            set -eu
                            git diff --check
                            echo "JSON changes:"
                            git diff -- $FILES_TO_REVIEW
                        '''
                    }

                    env.CHANGES_EXIST = sh(
                        script: 'git status --porcelain',
                        returnStdout: true
                    ).trim() ? 'true' : 'false'

                    if (env.CHANGES_EXIST == 'false') {
                        echo 'No changes detected. The pipeline will skip commit, push, and pull request creation.'
                    }
                }
            }
        }

        stage('Commit changes') {
            when {
                expression { env.CHANGES_EXIST == 'true' }
            }
            steps {
                sh '''
                    set -eu
                    git add "node/$SELECTED_ENV.json" "environment/$SELECTED_ENV.json"
                    git diff --cached --check
                    git commit -m "Update $SELECTED_ENV configuration"
                '''
            }
        }

        stage('Push feature branch') {
            when {
                expression { env.CHANGES_EXIST == 'true' }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.GIT_CREDENTIALS_ID,
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        ASKPASS_FILE="$(mktemp)"
                        trap 'rm -f "$ASKPASS_FILE"' EXIT

                        cat > "$ASKPASS_FILE" <<'ASKPASS'
#!/bin/sh
case "$1" in
    *Username*) printf '%s\\n' "$GIT_USERNAME" ;;
    *Password*) printf '%s\\n' "$GIT_PASSWORD" ;;
    *) exit 1 ;;
esac
ASKPASS
                        chmod 700 "$ASKPASS_FILE"

                        export GIT_ASKPASS="$ASKPASS_FILE"
                        export GIT_TERMINAL_PROMPT=0

                        git push origin "$FEATURE_BRANCH"
                    '''
                }
            }
        }

        stage('Create or find pull request') {
            when {
                expression { env.CHANGES_EXIST == 'true' }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.GIT_CREDENTIALS_ID,
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        python3 - <<'PY'
import json
import os
import urllib.error
import urllib.parse
import urllib.request

repository = os.environ["GITHUB_REPO"]
branch = os.environ["FEATURE_BRANCH"]
base = os.environ["BASE_BRANCH"]
token = os.environ["GITHUB_TOKEN"]

api = f"https://api.github.com/repos/{repository}/pulls"
query = urllib.parse.urlencode({
    "state": "open",
    "head": f"Vineeth7861:{branch}",
    "base": base,
})

def request(url, method="GET", payload=None):
    data = None if payload is None else json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(url, data=data, method=method)
    req.add_header("Authorization", f"Bearer {token}")
    req.add_header("Accept", "application/vnd.github+json")
    req.add_header("X-GitHub-Api-Version", "2022-11-28")
    if data is not None:
        req.add_header("Content-Type", "application/json")
    with urllib.request.urlopen(req, timeout=30) as response:
        return json.load(response)

try:
    existing = request(f"{api}?{query}")
    if existing:
        print(f"An open pull request already exists: {existing[0]['html_url']}")
    else:
        payload = {
            "title": f"Update {os.environ['SELECTED_ENV']} configuration",
            "head": branch,
            "base": base,
            "body": f"Configuration update created by Jenkins build {os.environ['BUILD_NUMBER']}."
        }
        created = request(api, method="POST", payload=payload)
        print(f"Pull request created: {created['html_url']}")
except urllib.error.HTTPError as error:
    details = error.read().decode("utf-8", errors="replace")
    print(f"GitHub API request failed with HTTP {error.code}: {details}")
    raise
PY
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            script {
                if (env.CHANGES_EXIST == 'false') {
                    echo 'No files changed; no commit, push, or pull request was made.'
                } else if (env.CHANGES_EXIST == 'true') {
                    echo "Configuration changes for ${env.SELECTED_ENV} were pushed from ${env.FEATURE_BRANCH}."
                    echo 'Check the Jenkins Console Output for the pull request link.'
                }
            }
        }

        failure {
            echo 'Pipeline failed. Open Console Output and look for the first error message.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
