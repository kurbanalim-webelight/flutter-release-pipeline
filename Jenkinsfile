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

        stage('Validate Inputs') {
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
                }
            }
        }

        stage('Load Configuration') {
            steps {
                script {
                    if (!fileExists(env.CONFIG_FILE)) {
                        error "Configuration file not found: ${env.CONFIG_FILE}"
                    }

                    projectConfig = parseProperties(readFile(env.CONFIG_FILE))
                    if (!projectConfig) {
                        error "No settings found in ${env.CONFIG_FILE}."
                    }

                    String flutterVersion = projectConfig.get('FLUTTER_VERSION')
                    if (!flutterVersion) {
                        error "FLUTTER_VERSION is not set in ${env.CONFIG_FILE}."
                    }
                    if (!(flutterVersion ==~ /^\d+\.\d+\.\d+(-\S+)?$/)) {
                        error "FLUTTER_VERSION must look like 3.35.4, got '${flutterVersion}'."
                    }

                    env.FLUTTER_VERSION = flutterVersion

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
                '''
            }
        }

        stage('Verify Flutter Version') {
            steps {
                sh '''
                    set -eu

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
