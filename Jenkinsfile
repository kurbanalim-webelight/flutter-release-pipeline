Map<String, String> projectConfig = [:]

pipeline {
    agent { label 'macos' }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        CONFIG_FILE = '/Users/macbook-pro-002/Desktop/release-pipeline/pipeline.properties'
    }

    parameters {
        choice(
            name: 'PLATFORM',
            choices: ['Android', 'iOS'],
            description: 'Target platform'
        )
        choice(
            name: 'BUILD_RUNNER',
            choices: ['Flutter', 'Shorebird'],
            description: 'Flutter for a normal build, Shorebird for a code push release'
        )
        string(
            name: 'BUILD_VERSION',
            defaultValue: '',
            trim: true,
            description: 'Semantic version, e.g. 1.0.0 (required)'
        )
        string(
            name: 'APP_BUILD_NUMBER',
            defaultValue: '',
            trim: true,
            description: 'Positive integer, e.g. 1, 10, 32 (required)'
        )
    }

    stages {

        stage('Validate & Load Configuration') {
            steps {
                script {
                    List<String> errors = []

                    if (!params.BUILD_VERSION) {
                        errors << 'BUILD_VERSION is required.'
                    } else if (!(params.BUILD_VERSION ==~ /^\d+\.\d+\.\d+$/)) {
                        errors << "BUILD_VERSION must be MAJOR.MINOR.PATCH, got '${params.BUILD_VERSION}'."
                    }

                    if (!params.APP_BUILD_NUMBER) {
                        errors << 'APP_BUILD_NUMBER is required.'
                    } else if (!(params.APP_BUILD_NUMBER ==~ /^\d+$/)) {
                        errors << "APP_BUILD_NUMBER must be a whole number, got '${params.APP_BUILD_NUMBER}'."
                    } else if (params.APP_BUILD_NUMBER.toInteger() < 1) {
                        errors << 'APP_BUILD_NUMBER must be 1 or greater.'
                    }

                    if (errors) {
                        error "Invalid build inputs:\n  - ${errors.join('\n  - ')}"
                    }

                    currentBuild.displayName =
                        "#${env.BUILD_NUMBER} ${params.PLATFORM}/${params.BUILD_RUNNER} " +
                        "${params.BUILD_VERSION}+${params.APP_BUILD_NUMBER}"

                    if (!fileExists(env.CONFIG_FILE)) {
                        error "Configuration file not found: ${env.CONFIG_FILE}"
                    }

                    projectConfig = parseProperties(readFile(env.CONFIG_FILE))
                    if (!projectConfig) {
                        error "No settings found in ${env.CONFIG_FILE}."
                    }

                    List<String> missing = []
                    ['FLUTTER_VERSION', 'GIT_REPO_URL', 'GIT_BRANCH', 'GIT_CREDENTIALS_ID'].each { String key ->
                        if (!projectConfig.get(key)) {
                            missing << key
                        }
                    }
                    if (missing) {
                        error "Missing in ${env.CONFIG_FILE}:\n  - ${missing.join('\n  - ')}"
                    }

                    String flutterVersion = projectConfig.get('FLUTTER_VERSION')
                    if (!(flutterVersion ==~ /^\d+\.\d+\.\d+(-\S+)?$/)) {
                        error "FLUTTER_VERSION must look like 3.35.4, got '${flutterVersion}'."
                    }

                    String repoUrl = projectConfig.get('GIT_REPO_URL')
                    if (!(repoUrl ==~ /^(https:\/\/|git@).+/)) {
                        error "GIT_REPO_URL must start with https:// or git@, got '${repoUrl}'."
                    }

                    String credentialsId = projectConfig.get('GIT_CREDENTIALS_ID')
                    if (credentialsId ==~ /^(glpat-|gh[pousr]_|xox[baprs]-).*/) {
                        error 'GIT_CREDENTIALS_ID looks like an access token, not a credential ID. ' +
                              'Store the token in Jenkins Credentials and put its ID here.'
                    }

                    env.FLUTTER_VERSION    = flutterVersion
                    env.GIT_REPO_URL       = repoUrl
                    env.GIT_BRANCH         = projectConfig.get('GIT_BRANCH')
                    env.GIT_CREDENTIALS_ID = credentialsId

                    echo "Loaded ${projectConfig.size()} setting(s) from ${env.CONFIG_FILE}"
                }
            }
        }

        stage('Setup Flutter') {
            steps {
                sh '''
                    set -eu

                    if ! command -v fvm >/dev/null 2>&1; then
                        echo "fvm not found on this agent - installing"
                        if ! command -v brew >/dev/null 2>&1; then
                            echo "Homebrew is required to install fvm. Install it on this agent." >&2
                            exit 1
                        fi
                        HOMEBREW_NO_AUTO_UPDATE=1 HOMEBREW_NO_INSTALL_CLEANUP=1 brew install fvm
                    fi
                    echo "fvm $(fvm --version)"

                    fvm install "$FLUTTER_VERSION" --setup
                    fvm use "$FLUTTER_VERSION" --force

                    ACTUAL=$(fvm flutter --version 2>/dev/null | awk '/^Flutter /{print $2; exit}')

                    if [ -z "$ACTUAL" ]; then
                        echo "Could not determine the active Flutter version" >&2
                        exit 1
                    fi

                    echo "expected: $FLUTTER_VERSION"
                    echo "actual:   $ACTUAL"

                    if [ "$ACTUAL" != "$FLUTTER_VERSION" ]; then
                        echo "Flutter version mismatch: expected $FLUTTER_VERSION but the active SDK is $ACTUAL" >&2
                        exit 1
                    fi

                    echo "Flutter version matches"
                '''
            }
        }

        stage('Clone Repository') {
            steps {
                checkout scmGit(
                    branches: [[name: "*/${env.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: env.GIT_REPO_URL,
                        credentialsId: env.GIT_CREDENTIALS_ID
                    ]],
                    extensions: [[$class: 'CloneOption', shallow: true, depth: 1, noTags: true]]
                )
                sh 'git --no-pager log -1 --format="%h %s"'
            }
        }
    }
}

Map<String, String> parseProperties(String text) {
    Map<String, String> result = [:]
    text.split('\\R').each { String line ->
        String entry = line.trim()
        if (!entry || entry.startsWith('#') || entry.startsWith('!')) {
            return
        }
        int separator = entry.indexOf('=')
        if (separator < 1) {
            return
        }
        result[entry.substring(0, separator).trim()] = entry.substring(separator + 1).trim()
    }
    return result
}
