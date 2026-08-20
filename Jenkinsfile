import groovy.transform.Field

// @Field promotes this to a binding property. A plain top-level declaration is a local
// of the script's run(), which the helper methods below cannot see.
@Field Map<String, String> projectConfig = [:]

pipeline {
    agent { label 'macos' }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        CONFIG_FILE = '/Users/macbook-pro-002/Desktop/release-pipeline/pipeline.properties'

        // Release builds always ship the prod flavour. Both CLIs spell these the same way.
        BUILD_FLAVOR    = 'prod'
        DART_ENTRYPOINT = 'lib/config/flavours/prod/prod.dart'
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
                    section('🔍  Validate & Load Configuration')

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

                    ['GIT_CREDENTIALS_ID', 'INFISICAL_TOKEN_CREDENTIALS_ID', 'S3_CREDENTIALS_ID',
                     'SHOREBIRD_TOKEN_CREDENTIALS_ID'].each { String key ->
                        if (projectConfig.get(key) ==~ /^(glpat-|gh[pousr]_|xox[baprs]-|st\.).*/) {
                            error "${key} looks like a secret, not a credential ID. " +
                                  'Store the secret in Jenkins Credentials and put its ID here.'
                        }
                    }
                    String credentialsId = projectConfig.get('GIT_CREDENTIALS_ID')

                    if (params.PLATFORM == 'iOS') {
                        List<String> missingAsc = ['ASC_KEY_ID', 'ASC_ISSUER_ID'].findAll { String key ->
                            !projectConfig.get(key)
                        }
                        if (missingAsc) {
                            error "Required in ${env.CONFIG_FILE} for an iOS build:\n  - " +
                                  missingAsc.join('\n  - ')
                        }
                        env.ASC_KEY_ID    = projectConfig.get('ASC_KEY_ID')
                        env.ASC_ISSUER_ID = projectConfig.get('ASC_ISSUER_ID')
                        env.ASC_KEY_FILE  = "private_keys/AuthKey_${projectConfig.get('ASC_KEY_ID')}.p8"

                        if (!projectConfig.get('SECRET_FILE.private_keys/AuthKey.p8')) {
                            error 'The TestFlight upload needs the App Store Connect key. Add ' +
                                  'SECRET_FILE.private_keys/AuthKey.p8=<s3 uri> to ' + env.CONFIG_FILE + '.'
                        }
                    }

                    if (params.PLATFORM == 'Android') {
                        String packageName = projectConfig.get('ANDROID_PACKAGE_NAME')
                        if (!packageName) {
                            error "ANDROID_PACKAGE_NAME is required in ${env.CONFIG_FILE} for an Android build."
                        }
                        if (!(packageName ==~ /^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)+$/)) {
                            error "ANDROID_PACKAGE_NAME must be a package id like com.example.app, got '${packageName}'."
                        }
                        env.ANDROID_PACKAGE_NAME = packageName
                        env.PLAY_KEY_FILE        = 'private_keys/play-store.json'

                        if (!projectConfig.get('SECRET_FILE.private_keys/play-store.json')) {
                            error 'The Play upload needs the service account key. Add ' +
                                  'SECRET_FILE.private_keys/play-store.json=<s3 uri> to ' +
                                  env.CONFIG_FILE + '.'
                        }
                    }

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

                    echo "✅  Inputs OK: ${params.PLATFORM} / ${params.BUILD_RUNNER} ${params.BUILD_VERSION}+${params.APP_BUILD_NUMBER}"
                    echo "📦  Repository: ${env.GIT_URL}"
                    echo "🌿  Branch:     ${env.GIT_BRANCH}"
                    echo "🐦  Flutter:    ${env.FLUTTER_VERSION}"
                    echo "🔐  Infisical:  ${env.INFISICAL_ENV} @ ${env.INFISICAL_API_URL}"
                    echo "⚙️   Settings:   ${projectConfig.size()} loaded from ${env.CONFIG_FILE}"
                    if (params.PLATFORM == 'iOS') {
                        echo "🚀  TestFlight: key ${env.ASC_KEY_ID}"
                    } else {
                        echo "🚀  Play:       ${env.ANDROID_PACKAGE_NAME} -> internal track"
                    }

                    if (params.BUILD_RUNNER == 'Shorebird') {
                        precheckShorebird()
                    }
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    section('📥  Clone Repository')

                    git branch: env.GIT_BRANCH,
                        credentialsId: env.GIT_CREDENTIALS_ID,
                        url: env.GIT_URL

                    String head = sh(
                        script: 'git --no-pager log -1 --format="%h %s"',
                        returnStdout: true
                    ).trim()
                    echo "🔖  HEAD: ${head}"
                }
            }
        }

        stage('Setup Flutter') {
            steps {
                script {
                    section('🐦  Setup Flutter')

                    setupFlutter()
                }
            }
        }

        stage('Load Secret Files') {
            steps {
                script {
                    section('🔐  Load Secret Files')

                    withCredentials([string(
                        credentialsId: env.INFISICAL_TOKEN_CREDENTIALS_ID,
                        variable: 'INFISICAL_TOKEN'
                    )]) {
                        exportEnvFile()
                    }

                    Map<String, String> secretFiles = [:]
                    projectConfig.each { String key, String value ->
                        if (key.startsWith('SECRET_FILE.')) {
                            secretFiles[key.substring('SECRET_FILE.'.length())] = value
                        }
                    }

                    if (!secretFiles) {
                        echo '📭  No SECRET_FILE.* entries configured - nothing to download.'
                        return
                    }

                    withCredentials([usernamePassword(
                        credentialsId: projectConfig.get('S3_CREDENTIALS_ID'),
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )]) {
                        downloadSecretFiles(
                            secretFiles,
                            projectConfig.get('S3_ENDPOINT'),
                            projectConfig.get('S3_REGION')
                        )
                    }

                    echo "✅  Downloaded ${secretFiles.size()} secret file(s)"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    section('📦  Install Dependencies')

                    installDependencies(params.PLATFORM == 'iOS')
                    generateCode()
                }
            }
        }

        stage('Build Release') {
            steps {
                script {
                    section('🏗️   Build Release')

                    if (params.PLATFORM == 'iOS') {
                        buildIOS()
                    } else {
                        buildAndroid()
                    }
                }
            }
        }

        stage('Upload Release') {
            steps {
                script {
                    if (params.PLATFORM == 'iOS') {
                        section('🚀  Upload to TestFlight')
                        uploadToTestFlight()
                    } else {
                        section('🚀  Upload to Play Internal Testing')
                        uploadToPlay()
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                String result = currentBuild.currentResult
                String badge = result == 'SUCCESS' ? '🎉' : (result == 'ABORTED' ? '🛑' : '💥')
                section("${badge}  Build ${result}")
                echo "🏷️   ${currentBuild.displayName}"
                echo "⏱️   Duration: ${currentBuild.durationString.replace(' and counting', '')}"
                echo "🪵  Console:  ${env.BUILD_URL}console"
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

void section(String title) {
    String rule = '━' * 66
    echo "\n${rule}\n  ${title}\n${rule}"
}

void ensureTool(String command, String formula) {
    sh """
        set -eu

        if ! command -v ${command} >/dev/null 2>&1; then
            echo "⬇️  ${command} not found on this agent - installing"
            if ! command -v brew >/dev/null 2>&1; then
                echo "❌  Homebrew is required to install ${formula}. Install it on this agent." >&2
                exit 1
            fi
            HOMEBREW_NO_AUTO_UPDATE=1 HOMEBREW_NO_INSTALL_CLEANUP=1 brew install ${formula}
        fi
        echo "🧰  ${command}: \$(${command} --version 2>&1 | head -1)"
    """
}

void setupFlutter() {
    ensureTool('fvm', 'fvm')
    sh '''
        set -eu

        fvm install "$FLUTTER_VERSION" --setup
        fvm use "$FLUTTER_VERSION" --force

        ACTUAL=$(fvm flutter --version 2>/dev/null | awk '/^Flutter /{print $2; exit}')

        if [ -z "$ACTUAL" ]; then
            echo "❌  Could not determine the active Flutter version" >&2
            exit 1
        fi

        echo "🎯  expected: $FLUTTER_VERSION"
        echo "📍  actual:   $ACTUAL"

        if [ "$ACTUAL" != "$FLUTTER_VERSION" ]; then
            echo "❌  Flutter version mismatch: expected $FLUTTER_VERSION but the active SDK is $ACTUAL" >&2
            exit 1
        fi

        echo "✅  Flutter version matches"
    '''
}

void installDependencies(boolean withPods) {
    sh '''
        set -eu

        fvm flutter clean
        fvm flutter pub get
    '''

    if (!withPods) {
        echo '💭  Android build - skipping pod install.'
        return
    }

    ensureTool('pod', 'cocoapods')
    sh '''
        set -eu

        fvm flutter precache --ios
        cd ios

        rm -rf Pods
        rm -f Podfile.lock
        pod setup

        pod install --repo-update
    '''
}

void exportEnvFile() {
    ensureTool('infisical', 'infisical')
    sh '''
        set -eu

        infisical export \
            --projectId="$INFISICAL_PROJECT_ID" \
            --env="$INFISICAL_ENV" \
            --domain="$INFISICAL_API_URL" \
            --format=dotenv > .env

        if [ ! -s .env ]; then
            echo "❌  Infisical returned nothing - .env is empty" >&2
            exit 1
        fi

        # Infisical emits KEY='value'; envied does not strip the quotes, so an
        # empty secret reaches Dart as the literal two-character string ''.
        sed -E "s/^([A-Za-z_][A-Za-z0-9_]*)='(.*)'\$/\\1=\\2/" .env > .env.tmp
        mv .env.tmp .env

        echo "🗝️   .env written to $(pwd) with $(grep -c "^[A-Za-z_]" .env) variable(s)"
    '''
}

void downloadSecretFiles(Map<String, String> secretFiles, String endpoint, String region) {
    ensureTool('aws', 'awscli')

    sh 'echo "🔑  storage access key id: ${#AWS_ACCESS_KEY_ID} characters"'

    secretFiles.each { String destination, String source ->
        sh """
            set -eu

            mkdir -p "\$(dirname '${destination}')"
            AWS_DEFAULT_REGION='${region}' \\
            AWS_REQUEST_CHECKSUM_CALCULATION=when_required \\
            AWS_RESPONSE_CHECKSUM_VALIDATION=when_required \\
                aws s3 cp '${source}' '${destination}' --endpoint-url '${endpoint}'

            if [ ! -s '${destination}' ]; then
                echo "❌  Downloaded file is empty: ${destination}" >&2
                exit 1
            fi

            echo "⬇️   ${destination} <- ${source}"
        """
    }
}

void precheckShorebird() {
    String credentialsId = projectConfig.get('SHOREBIRD_TOKEN_CREDENTIALS_ID')
    if (!credentialsId) {
        error "SHOREBIRD_TOKEN_CREDENTIALS_ID is required in ${env.CONFIG_FILE} " +
              'when the build runner is Shorebird.'
    }

    withCredentials([string(credentialsId: credentialsId, variable: 'SHOREBIRD_TOKEN')]) {
        sh '''
            set -eu

            if [ -z "$SHOREBIRD_TOKEN" ]; then
                echo "❌  The Shorebird token credential is empty" >&2
                exit 1
            fi
            echo "🔑  shorebird token: ${#SHOREBIRD_TOKEN} characters"

            if ! [ -x "$HOME/.shorebird/bin/shorebird" ]; then
                echo "⬇️  Shorebird not found on this agent - installing"
                curl -fsSL https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh | bash
            fi

            export PATH="$HOME/.shorebird/bin:$PATH"
            echo "🧰  shorebird: $(shorebird --version 2>&1 | head -1)"
        '''
    }
}

void generateCode() {
    sh '''
        set -eu

        # *.g.dart is git-ignored, so a fresh clone ships none of it and the build
        # will not compile without this step. Envied reads the .env written by the
        # Load Secret Files stage, which is why codegen has to run after it.
        fvm dart run easy_localization:generate \
            -S assets/l10n -s en.json -f keys -O lib/l10n -o locale_keys.g.dart
        fvm dart run build_runner build --delete-conflicting-outputs

        GENERATED=$(find lib -name "*.g.dart" | wc -l | tr -d ' ')

        if [ "$GENERATED" = "0" ]; then
            echo "❌  Code generation produced no *.g.dart files" >&2
            exit 1
        fi

        echo "🧬  $GENERATED generated file(s) under lib/"
    '''
}

void buildIOS() {
    // Signing is Automatic in Runner.xcodeproj with the team baked in, so the only
    // requirement is that the agent's Xcode is signed in to the Apple account by hand.
    if (params.BUILD_RUNNER != 'Shorebird') {
        flutterBuildIOS()
        return
    }

    withCredentials([string(
        credentialsId: projectConfig.get('SHOREBIRD_TOKEN_CREDENTIALS_ID'),
        variable: 'SHOREBIRD_TOKEN'
    )]) {
        shorebirdReleaseIOS()
    }
}

void buildAndroid() {
    ensureGradleJitpackFix()

    if (params.BUILD_RUNNER != 'Shorebird') {
        flutterBuildAndroid()
        return
    }

    withCredentials([string(
        credentialsId: projectConfig.get('SHOREBIRD_TOKEN_CREDENTIALS_ID'),
        variable: 'SHOREBIRD_TOKEN'
    )]) {
        shorebirdReleaseAndroid()
    }
}

void ensureGradleJitpackFix() {
    // image_cropper 9.1.0 declares "https://www.jitpack.io", whose IPv4 address serves a
    // GitHub certificate, so Gradle cannot fetch com.github.yalantis:ucrop. The apex host
    // is healthy and is what image_cropper itself moved to in 11.0.0. Rewriting the URL in
    // a Gradle init script fixes every Gradle build on the agent without touching the app
    // repo. Drop this once the app upgrades image_cropper past 11.0.0.
    sh '''
        set -eu

        mkdir -p "$HOME/.gradle/init.d"
        cat > "$HOME/.gradle/init.d/jitpack-www.gradle" <<'INIT'
allprojects {
    buildscript.repositories.withType(MavenArtifactRepository) { repo ->
        if (repo.url.toString().startsWith('https://www.jitpack.io')) { repo.url = 'https://jitpack.io' }
    }
    repositories.withType(MavenArtifactRepository) { repo ->
        if (repo.url.toString().startsWith('https://www.jitpack.io')) { repo.url = 'https://jitpack.io' }
    }
}
INIT

        echo "🩹  jitpack repository fix installed for Gradle"
    '''
}

void flutterBuildAndroid() {
    sh '''
        set -eu

        fvm flutter build appbundle \
            --release \
            --flavor "$BUILD_FLAVOR" \
            -t "$DART_ENTRYPOINT" \
            --build-name="$BUILD_VERSION" \
            --build-number="$APP_BUILD_NUMBER"

        AAB=$(find build/app/outputs/bundle -name "*.aab" -print -quit 2>/dev/null || true)

        if [ -z "$AAB" ]; then
            echo "❌  No .aab found under build/app/outputs/bundle" >&2
            exit 1
        fi

        echo "📦  $AAB ($(du -h "$AAB" | cut -f1))"
    '''
}

void shorebirdReleaseAndroid() {
    sh '''
        set -eu

        export PATH="$HOME/.shorebird/bin:$PATH"

        shorebird release -p android \
            --flavor "$BUILD_FLAVOR" \
            -t "$DART_ENTRYPOINT" \
            --artifact aab \
            --build-name="$BUILD_VERSION" \
            --build-number="$APP_BUILD_NUMBER"
    '''
}

void flutterBuildIOS() {
    sh '''
        set -eu

        fvm flutter build ipa \
            --release \
            --flavor "$BUILD_FLAVOR" \
            -t "$DART_ENTRYPOINT" \
            --build-name="$BUILD_VERSION" \
            --build-number="$APP_BUILD_NUMBER" \
            --export-method app-store
    '''

    reportIpa()
}

void shorebirdReleaseIOS() {
    sh '''
        set -eu

        export PATH="$HOME/.shorebird/bin:$PATH"

        shorebird release -p ios \
            --flavor "$BUILD_FLAVOR" \
            -t "$DART_ENTRYPOINT" \
            --build-name="$BUILD_VERSION" \
            --build-number="$APP_BUILD_NUMBER" \
            --export-method app-store
    '''

    reportIpa()
}

void uploadToTestFlight() {
    sh '''
        set -eu

        IPA=$(find build/ios/ipa -name "*.ipa" -print -quit 2>/dev/null || true)

        if [ -z "$IPA" ]; then
            echo "❌  No .ipa found under build/ios/ipa" >&2
            exit 1
        fi

        # The key travels under a generic name so the storage object never has to be
        # renamed when the key rotates. altool only reads AuthKey_<key id>.p8, from
        # private_keys/ next to the working directory, so copy it into place.
        if [ ! -s private_keys/AuthKey.p8 ]; then
            echo "❌  private_keys/AuthKey.p8 is missing - the secret file did not download" >&2
            exit 1
        fi
        cp private_keys/AuthKey.p8 "$ASC_KEY_FILE"

        echo "📤  Uploading $IPA ($(du -h "$IPA" | cut -f1)) with key $ASC_KEY_ID"

        xcrun altool --upload-app \
            --type ios \
            --file "$IPA" \
            --apiKey "$ASC_KEY_ID" \
            --apiIssuer "$ASC_ISSUER_ID"

        echo "✅  Uploaded to TestFlight - Apple processes the build before it appears"
    '''
}

void uploadToPlay() {
    ensureTool('fastlane', 'fastlane')
    sh '''
        set -eu

        # fastlane refuses to run under a non-UTF-8 locale, which is what a Jenkins
        # agent shell gets by default.
        export LC_ALL=en_US.UTF-8
        export LANG=en_US.UTF-8

        AAB=$(find build/app/outputs/bundle -name "*.aab" -print -quit 2>/dev/null || true)

        if [ -z "$AAB" ]; then
            echo "❌  No .aab found under build/app/outputs/bundle" >&2
            exit 1
        fi

        # Same handling as the App Store Connect key: the service account JSON comes
        # down under a generic name, so rotating the key never touches this pipeline.
        if [ ! -s "$PLAY_KEY_FILE" ]; then
            echo "❌  $PLAY_KEY_FILE is missing - the secret file did not download" >&2
            exit 1
        fi

        echo "📤  Uploading $AAB ($(du -h "$AAB" | cut -f1)) to $ANDROID_PACKAGE_NAME"

        fastlane supply \
            --aab "$AAB" \
            --package_name "$ANDROID_PACKAGE_NAME" \
            --json_key "$PLAY_KEY_FILE" \
            --track internal \
            --release_status completed \
            --skip_upload_metadata \
            --skip_upload_images \
            --skip_upload_screenshots \
            --skip_upload_changelogs

        echo "✅  Uploaded to internal testing - Google processes the build before testers see it"
    '''
}

void reportIpa() {
    sh '''
        set -eu

        IPA=$(find build/ios/ipa -name "*.ipa" -print -quit 2>/dev/null || true)

        if [ -z "$IPA" ]; then
            echo "❌  No .ipa found under build/ios/ipa" >&2
            exit 1
        fi

        echo "📦  $IPA ($(du -h "$IPA" | cut -f1))"
    '''
}
