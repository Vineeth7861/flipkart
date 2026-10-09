pipeline {
    agent any
    options {
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timestamps()
    }
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Select environment'
        )
        choice(
            name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Select configuration type'
        )
        string(
            name: 'INSTANCE_TYPE',
            defaultValue: '',
            description: 'Mandatory for node updates'
        )
        string(
            name: 'K8S_VERSION',
            defaultValue: '',
            description: 'Optional: Kubernetes version'
        )
        string(
            name: 'CPU',
            defaultValue: '',
            description: 'Optional: Node CPU'
        )
        string(
            name: 'MEMORY',
            defaultValue: '',
            description: 'Optional: Node memory'
        )
        string(
            name: 'ENV_APPNAME',
            defaultValue: '',
            description: 'Mandatory application name'
        )
        string(
            name: 'ENV_VERSION',
            defaultValue: '',
            description: 'Mandatory version, e.g. 2.0.0'
        )
        string(
            name: 'ENV_REPLICAS',
            defaultValue: '',
            description: 'Mandatory replicas, e.g. 2'
        )
        choice(
            name: 'ENV_LOGLEVEL',
            choices: ['KEEP', 'DEBUG', 'INFO', 'WARN', 'ERROR'],
            description: 'KEEP preserves existing log level'
        )
    }
    environment {
        HAS_CHANGES = 'false'
        FEATURE_BRANCH = ''
        GITHUB_REPO = 'chandrashekar0532/flipkart'
    }
    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
                sh '''
                    git fetch origin main
                    git checkout -B main origin/main
                    git config user.name "Jenkins Automation"
                    git config user.email "jenkins-automation@example.com"
                '''
            }
        }
        stage('Display Parameters') {
            steps {
                echo "Environment: ${params.ENVIRONMENT}"
                echo "Config Type: ${params.CONFIG_TYPE}"
            }
        }
        stage('Validate Parameters') {
            steps {
                script {
                    if (!(params.ENVIRONMENT in
                        ['dev', 'stage', 'uat', 'prod'])) {
                        error('Invalid ENVIRONMENT')
                    }
                    if (!(params.CONFIG_TYPE in
                        ['both', 'environment', 'node'])) {
                        error('Invalid CONFIG_TYPE')
                    }
                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        if (!params.INSTANCE_TYPE?.trim()) {
                            error('INSTANCE_TYPE is mandatory')
                        }
                        if (!(params.INSTANCE_TYPE.trim() ==~
                            /[a-z][a-z0-9]*[0-9][a-z0-9-]*\.[a-z0-9]+/)) {
                            error('Invalid INSTANCE_TYPE format')
                        }
                        if (params.CPU?.trim() &&
                            !(params.CPU.trim() ==~ /[1-9][0-9]*/)) {
                            error('CPU must be a positive integer')
                        }
                        if (params.MEMORY?.trim() &&
                            !(params.MEMORY.trim() ==~
                              /[1-9][0-9]*(Mi|Gi)/)) {
                            error('Invalid MEMORY format')
                        }
                    }
                    if (params.CONFIG_TYPE in
                        ['environment', 'both']) {
                        if (!params.ENV_APPNAME?.trim()) {
                            error('ENV_APPNAME is mandatory')
                        }
                        if (!params.ENV_VERSION?.trim()) {
                            error('ENV_VERSION is mandatory')
                        }
                        if (!(params.ENV_VERSION.trim() ==~
                            /[0-9]+\.[0-9]+\.[0-9]+/)) {
                            error('ENV_VERSION must be like 2.0.0')
                        }
                        if (!params.ENV_REPLICAS?.trim()) {
                            error('ENV_REPLICAS is mandatory')
                        }
                        if (!(params.ENV_REPLICAS.trim() ==~
                            /[1-9][0-9]*/)) {
                            error('Replicas must be positive')
                        }
                    }
                    echo 'All mandatory parameters validated'
                }
            }
        }
        stage('Read Configuration Files') {
            steps {
                script {
                    def types = params.CONFIG_TYPE == 'both'
                        ? ['environment', 'node']
                        : [params.CONFIG_TYPE]
                    types.each { type ->
                        def path =
                            "${type}/${params.ENVIRONMENT}.json"
                        if (!fileExists(path)) {
                            error("Missing JSON file: ${path}")
                        }
                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )
                        if (!(config instanceof Map)) {
                            error("Invalid JSON object: ${path}")
                        }
                        echo "Validated JSON: ${path}"
                    }
                }
            }
        }
        stage('Update JSON Configuration') {
            steps {
                script {
                    def types = params.CONFIG_TYPE == 'both'
                        ? ['environment', 'node']
                        : [params.CONFIG_TYPE]
                    types.each { type ->
                        def path =
                            "${type}/${params.ENVIRONMENT}.json"
                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )
                        def original = readJSON(
                            file: path,
                            returnPojo: true
                        )
                        if (type == 'environment') {
                            config.appName =
                                params.ENV_APPNAME.trim()
                            config.version =
                                params.ENV_VERSION.trim()
                            config.replicas =
                                params.ENV_REPLICAS.trim().toInteger()
                            if (params.ENV_LOGLEVEL != 'KEEP') {
                                config.logLevel =
                                    params.ENV_LOGLEVEL
                            }
                            echo "Application: ${config.appName}"
                            echo "Version: ${config.version}"
                            echo "Replicas: ${config.replicas}"
                            echo "Log level: ${config.logLevel}"
                        }
                        if (type == 'node') {
                            config.instanceType =
                                params.INSTANCE_TYPE.trim()
                            if (params.K8S_VERSION?.trim()) {
                                if (!(config.kubernetes instanceof Map)) {
                                    error("Missing kubernetes in ${path}")
                                }
                                config.kubernetes.version =
                                    params.K8S_VERSION.trim()
                            }
                            if (params.CPU?.trim()) {
                                if (!(config.resources instanceof Map)) {
                                    error("Missing resources in ${path}")
                                }
                                config.resources.cpu =
                                    params.CPU.trim()
                            }
                            if (params.MEMORY?.trim()) {
                                if (!(config.resources instanceof Map)) {
                                    error("Missing resources in ${path}")
                                }
                                config.resources.memory =
                                    params.MEMORY.trim()
                            }
                            echo "Instance Type: ${config.instanceType}"
                        }
                        if (config != original) {
                            writeJSON(
                                file: path,
                                json: config,
                                pretty: 4
                            )
                            echo "Updated JSON: ${path}"
                        } else {
                            echo "No value changes: ${path}"
                        }
                    }
                }
            }
        }
        stage('Verify Updated JSON') {
            steps {
                script {
                    if (params.CONFIG_TYPE in
                        ['environment', 'both']) {
                        def config = readJSON(
                            file: "environment/${params.ENVIRONMENT}.json",
                            returnPojo: true
                        )
                        echo "Verified App: ${config.appName}"
                        echo "Verified Version: ${config.version}"
                        echo "Verified Replicas: ${config.replicas}"
                        echo "Verified Log Level: ${config.logLevel}"
                    }
                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def config = readJSON(
                            file: "node/${params.ENVIRONMENT}.json",
                            returnPojo: true
                        )
                        echo "Verified Instance: ${config.instanceType}"
                        echo "Verified K8S: ${config.kubernetes?.version}"
                        echo "Verified CPU: ${config.resources?.cpu}"
                        echo "Verified Memory: ${config.resources?.memory}"
                    }
                }
            }
        }
        stage('Review Changes') {
            steps {
                script {
                    def paths = params.CONFIG_TYPE == 'both'
                        ? [
                            "environment/${params.ENVIRONMENT}.json",
                            "node/${params.ENVIRONMENT}.json"
                          ]
                        : [
                            "${params.CONFIG_TYPE}/${params.ENVIRONMENT}.json"
                          ]
                    sh 'git diff --check'
                    paths.each { path ->
                        sh "git diff -- '${path}'"
                    }
                    env.HAS_CHANGES = 'false'
                    paths.each { path ->
                        def status = sh(
                            script: "git diff --quiet -- '${path}'",
                            returnStatus: true
                        )
                        if (status == 1) {
                            env.HAS_CHANGES = 'true'
                        } else if (status != 0) {
                            error("Git diff failed for ${path}")
                        }
                    }
                    echo "Changes detected: ${env.HAS_CHANGES}"
                }
            }
        }
        stage('Create Git Feature Branch') {
            when {
                expression {
                    env.HAS_CHANGES == 'true'
                }
            }
            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"
                    sh 'git switch -c "$FEATURE_BRANCH"'
                    def paths = params.CONFIG_TYPE == 'both'
                        ? [
                            "environment/${params.ENVIRONMENT}.json",
                            "node/${params.ENVIRONMENT}.json"
                          ]
                        : [
                            "${params.CONFIG_TYPE}/${params.ENVIRONMENT}.json"
                          ]
                    paths.each { path ->
                        sh "git add -- '${path}'"
                    }
                    sh '''
                        git diff --cached --check
                        git commit -m "Update configuration via Jenkins"
                    '''
                    echo "Branch: ${env.FEATURE_BRANCH}"
                }
            }
        }
        stage('Push Feature Branch') {
            when {
                expression {
                    env.HAS_CHANGES == 'true'
                }
            }
            steps {
                withCredentials([
                    gitUsernamePassword(
                        credentialsId: 'github-private-creds',
                        gitToolName: 'Default'
                    )
                ]) {
                    sh 'git push -u origin "$FEATURE_BRANCH"'
                }
            }
        }
        stage('Create GitHub Pull Request') {
            when {
                expression {
                    env.HAS_CHANGES == 'true'
                }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-private-creds',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        python3 - <<'PY'
import json
import os
import urllib.request
import urllib.error
repo = os.environ["GITHUB_REPO"]
branch = os.environ["FEATURE_BRANCH"]
token = os.environ["GITHUB_TOKEN"]
payload = {
    "title": f"Update configuration: {branch}",
    "head": branch,
    "base": "main",
    "body": (
        "Automated configuration update from Jenkins. "
        "Review changes before merging."
    )
}
url = f"https://api.github.com/repos/{repo}/pulls"
request = urllib.request.Request(
    url,
    data=json.dumps(payload).encode("utf-8"),
    headers={
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28",
        "Content-Type": "application/json"
    },
    method="POST"
)
try:
    with urllib.request.urlopen(request, timeout=30) as response:
        result = json.load(response)
        print("Pull Request created:", result["html_url"])
except urllib.error.HTTPError as exc:
    print("GitHub PR creation failed:", exc.code)
    raise
PY
                    '''
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline completed successfully'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
        always {
            echo "Build Result: ${currentBuild.currentResult}"
        }
    }
}
