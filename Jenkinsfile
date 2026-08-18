/*
 * Step 3 - user inputs plus externalised per-project configuration.
 *
 * The Jenkinsfile is identical across projects. Anything that differs per
 * project lives in pipeline.properties, which the pipeline picks up on its own.
 *
 * Parsing is plain Groovy rather than readProperties/readYaml, so the pipeline
 * has no plugin prerequisites beyond Pipeline itself.
 */

// Populated by the Load Configuration stage, read by later stages.
Map<String, String> projectConfig = [:]

pipeline {
    agent { label 'macos' }

    options {
        timestamps()
        timeout(time: 5, unit: 'MINUTES')
    }

    environment {
        // Picked up automatically - deliberately not a parameter, since the user
        // has no reason to choose it. Becomes the workspace-relative
        // 'pipeline.properties' once a Checkout stage clones the project.
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
        // Not BUILD_NUMBER: Jenkins already uses that name for the run's own
        // number, and a parameter would shadow it.
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
                    // Collect every problem before failing, so one run reports
                    // everything that is wrong rather than one thing at a time.
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

                    // Exported explicitly so shell steps can use $FLUTTER_VERSION.
                    // Assigning env by subscript is blocked by the Groovy sandbox.
                    env.FLUTTER_VERSION = flutterVersion

                    echo "Loaded ${projectConfig.size()} setting(s) from ${env.CONFIG_FILE}"
                }
            }
        }

        stage('Show Configuration') {
            steps {
                script {
                    // Width comes from the longest key, so the columns stay lined
                    // up as new settings are added.
                    int width = 0
                    projectConfig.each { String key, String value ->
                        if (key.length() > width) {
                            width = key.length()
                        }
                    }
                    width += 2

                    String settings = projectConfig.collect { String key, String value ->
                        '  ' + (key + ':').padRight(width) + value
                    }.join('\n')

                    echo """
Build inputs (from the user)
----------------------------
  Platform:         ${params.PLATFORM}
  Build runner:     ${params.BUILD_RUNNER}
  Build version:    ${params.BUILD_VERSION}
  Build number:     ${params.APP_BUILD_NUMBER}
  Full version:     ${params.BUILD_VERSION}+${params.APP_BUILD_NUMBER}

Project configuration
---------------------
  source: ${env.CONFIG_FILE}

${settings}
"""
                }
            }
        }
    }
}

/**
 * Minimal .properties reader: KEY=VALUE per line, # or ! starts a comment.
 * Deliberately not java.util.Properties, which the Groovy sandbox blocks.
 */
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
