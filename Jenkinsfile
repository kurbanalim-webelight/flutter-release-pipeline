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
                    ['FLUTTER_VERSION', 'GIT_PROTOCOL', 'GIT_HOST', 'GIT_REPO_PATH', 'GIT_BRANCH',
                     'GIT_CREDENTIALS_ID', 'INFISICAL_API_URL', 'INFISICAL_PROJECT_ID',
                     'INFISICAL_ENV', 'INFISICAL_TOKEN_CREDENTIALS_ID'].each { String key ->
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

                    String repoPath = projectConfig.get('GIT_REPO_PATH')
                    if (repoPath.startsWith('/') || repoPath.endsWith('.git')) {
                        error "GIT_REPO_PATH must have no leading slash and no .git suffix, got '${repoPath}'."
                    }

                    ['GIT_CREDENTIALS_ID', 'INFISICAL_TOKEN_CREDENTIALS_ID', 'S3_CREDENTIALS_ID'].each { String key ->
                        if (projectConfig.get(key) ==~ /^(glpat-|gh[pousr]_|xox[baprs]-|st\.).*/) {
                            error "${key} looks like a secret, not a credential ID. " +
                                  'Store the secret in Jenkins Credentials and put its ID here.'
                        }
                    }
                    String credentialsId = projectConfig.get('GIT_CREDENTIALS_ID')

                    if (projectConfig.any { String key, String value -> key.startsWith('SECRET_FILE.') }) {
                        List<String> missingStorage =
                            ['S3_ENDPOINT', 'S3_REGION', 'S3_CREDENTIALS_ID'].findAll { String key ->
                                !projectConfig.get(key)
                            }
                        if (missingStorage) {
                            error "Required in ${env.CONFIG_FILE} when SECRET_FILE.* entries are set:\n  - " +
                                  missingStorage.join('\n  - ')
                        }
                    }

                    env.FLUTTER_VERSION    = flutterVersion
                    env.GIT_BRANCH         = projectConfig.get('GIT_BRANCH')
                    env.GIT_CREDENTIALS_ID = credentialsId
                    String protocol = projectConfig.get('GIT_PROTOCOL')
                    String host = projectConfig.get('GIT_HOST')
                    if (protocol == 'ssh') {
                        env.GIT_URL = 'git@' + host + ':' + repoPath + '.git'
                    } else if (protocol == 'https') {
                        env.GIT_URL = 'https://' + host + '/' + repoPath + '.git'
                    } else {
                        error "GIT_PROTOCOL must be ssh or https, got '${protocol}'."
                    }

                    env.INFISICAL_API_URL              = projectConfig.get('INFISICAL_API_URL')
                    env.INFISICAL_PROJECT_ID           = projectConfig.get('INFISICAL_PROJECT_ID')
                    env.INFISICAL_ENV                  = projectConfig.get('INFISICAL_ENV')
                    env.INFISICAL_TOKEN_CREDENTIALS_ID = projectConfig.get('INFISICAL_TOKEN_CREDENTIALS_ID')

                    echo "Repository: ${env.GIT_URL}"

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
                git branch: env.GIT_BRANCH,
                    credentialsId: env.GIT_CREDENTIALS_ID,
                    url: env.GIT_URL

                sh 'git --no-pager log -1 --format="%h %s"'
            }
        }

        stage('Load Secret Files') {
            steps {
                script {
                    withCredentials([string(
                        credentialsId: env.INFISICAL_TOKEN_CREDENTIALS_ID,
                        variable: 'INFISICAL_TOKEN'
                    )]) {
                        sh '''
                            set -eu

                            if ! command -v infisical >/dev/null 2>&1; then
                                echo "infisical not found on this agent - installing"
                                if ! command -v brew >/dev/null 2>&1; then
                                    echo "Homebrew is required to install the Infisical CLI." >&2
                                    exit 1
                                fi
                                HOMEBREW_NO_AUTO_UPDATE=1 HOMEBREW_NO_INSTALL_CLEANUP=1 brew install infisical
                            fi
                            echo "infisical $(infisical --version 2>&1 | head -1)"

                            infisical export \
                                --projectId="$INFISICAL_PROJECT_ID" \
                                --env="$INFISICAL_ENV" \
                                --domain="$INFISICAL_API_URL" \
                                --format=dotenv > .env

                            if [ ! -s .env ]; then
                                echo "Infisical returned nothing - .env is empty" >&2
                                exit 1
                            fi

                            echo ".env written to $(pwd) with $(grep -c "^[A-Za-z_]" .env) variable(s)"
                        '''
                    }

                    Map<String, String> secretFiles = [:]
                    projectConfig.each { String key, String value ->
                        if (key.startsWith('SECRET_FILE.')) {
                            secretFiles[key.substring('SECRET_FILE.'.length())] = value
                        }
                    }

                    if (!secretFiles) {
                        echo 'No SECRET_FILE.* entries configured - nothing to download.'
                        return
                    }

                    sh '''
                        set -eu

                        if ! command -v aws >/dev/null 2>&1; then
                            echo "aws not found on this agent - installing"
                            if ! command -v brew >/dev/null 2>&1; then
                                echo "Homebrew is required to install the aws CLI." >&2
                                exit 1
                            fi
                            HOMEBREW_NO_AUTO_UPDATE=1 HOMEBREW_NO_INSTALL_CLEANUP=1 brew install awscli
                        fi
                        echo "$(aws --version 2>&1 | head -1)"
                    '''

                    String endpoint = projectConfig.get('S3_ENDPOINT')
                    String region = projectConfig.get('S3_REGION')

                    withCredentials([usernamePassword(
                        credentialsId: projectConfig.get('S3_CREDENTIALS_ID'),
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )]) {
                        // R2 access key IDs are 32 characters. Pasting the API token
                        // instead gives a longer one and a bare 400 from the service.
                        sh 'echo "storage access key id: ${#AWS_ACCESS_KEY_ID} characters"'

                        secretFiles.each { String destination, String source ->
                            sh """
                                set -eu

                                mkdir -p "\$(dirname '${destination}')"
                                # Checksum headers are sent by default since aws-cli 2.23 and are
                                # rejected by some S3-compatible services. Ask for them only when
                                # the operation requires them.
                                AWS_DEFAULT_REGION='${region}' \\
                                AWS_REQUEST_CHECKSUM_CALCULATION=when_required \\
                                AWS_RESPONSE_CHECKSUM_VALIDATION=when_required \\
                                    aws s3 cp '${source}' '${destination}' --endpoint-url '${endpoint}'

                                if [ ! -s '${destination}' ]; then
                                    echo "Downloaded file is empty: ${destination}" >&2
                                    exit 1
                                fi

                                echo "${destination} <- ${source}"
                            """
                        }
                    }

                    echo "Downloaded ${secretFiles.size()} secret file(s)"
                }
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
