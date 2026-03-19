// https://github.com/Rudd-O/shared-jenkins-libraries
@Library('shared-jenkins-libraries@master') _

def esphome_version = "403ba262c683dab7266d54ed56807ad6074d8718";

pipeline {

        agent { label 'master' }

        options {
                skipDefaultCheckout()
                buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '1'))
        }

        stages {
                stage('Begin') {
                        steps {
                                announceBeginning()
                                script {
                                        funcs.durable()
                                }
                        }
                }
                stage('Checkout') {
                        steps {
                                dir('src') {
                                        checkout([
                                                $class: 'GitSCM',
                                                branches: scm.branches,
                                                extensions: [
                                                        [$class: 'CleanBeforeCheckout'],
                                                        [
                                                                $class: 'SubmoduleOption',
                                                                disableSubmodules: false,
                                                                parentCredentials: false,
                                                                recursiveSubmodules: true,
                                                                trackingSubmodules: false
                                                        ],
                                                ],
                                                userRemoteConfigs: scm.userRemoteConfigs
                                        ])
                                }
                                script {
                                        env.BUILD_DATE = sh(
                                                script: "date +%Y.%m.%d",
                                                returnStdout: true
                                        ).trim()
                                        env.BUILD_SRC_SHORT_COMMIT = sh(
                                                script: "cd src && git rev-parse --short HEAD",
                                                returnStdout: true
                                        ).trim()
                                        env.BUILD_UPSTREAM_SHORT_COMMIT = sh(
                                                script: '''
                                                        for a in upstream/*
                                                        do
                                                                if test -d "$a" && test -d "$a"/.git
                                                                then
                                                                        oldpwd=$(pwd)
                                                                        cd "$a"
                                                                        git rev-parse --short HEAD
                                                                        cd "$oldpwd"
                                                                fi
                                                        done
                                                ''',
                                                returnStdout: true
                                        ).trim()
                                }
                                updateBuildNumberDisplayName()
                                dir('src') {
                                        stash includes: '**', name: 'source', useDefaultExcludes: false
                                }
                        }
                }
                stage('Dispatch') {
                        agent { label 'esphome' }
                        environment {
                                PIP_CACHE_DIR = "${env.WORKSPACE}/../../caches/pip"
                        }
                        stages {
                                stage('Deps') {
                                        steps {
                                                script {
                                                        funcs.dnfInstall(['yq', 'jq'])
                                                }
                                                sh("""
                                                set -e
                                                if test -d esphome
                                                then
                                                        cd esphome
                                                        git fetch
                                                else
                                                        git clone https://github.com/esphome/esphome
                                                        cd esphome
                                                fi
                                                git checkout ${esphome_version}
                                                script/setup
                                                """)
                                        }
                                }
                                stage('Unstash') {
                                        steps {
                                                dir('src') {
                                                        deleteDir()
                                                        unstash 'source'
                                                }
                                        }
                                }
                                stage('Build') {
                                        steps {
                                                sh("""
                                                cp -f src/Code/digital-audio-bridge.yaml digital-audio-bridge.yaml
                                                source esphome/venv/bin/activate
                                                esphome compile digital-audio-bridge.yaml
                                                cp -f .esphome/build/digital-audio-bridge/.pioenvs/digital-audio-bridge/firmware.factory.bin firmware.factory.bin
                                                cp -f .esphome/build/digital-audio-bridge/.pioenvs/digital-audio-bridge/firmware.ota.bin firmware.ota.bin
                                                """)
                                        }
                                }
                                stage('Generate changelog') {
                                        steps {
                                                script {
                                                        dir('src') {
                                                                def tagName = sh(
                                                                        script: 'git describe --tags --exact-match 2>/dev/null || true',
                                                                        returnStdout: true
                                                                ).trim()
                                                                def lastTag = ""
                                                                if (tagName != "") {
                                                                        lastTag = sh(
                                                                                script: "git log --decorate=full | grep '^commit.*tag: refs/tags/' | head -2 | tail -1 | sed -r 's|.*refs/tags/|\1|' | sed -r 's|[,)].*||'",
                                                                                returnStdout: true
                                                                        ).trim()
                                                                } else {
                                                                        lastTag = sh(
                                                                                script: "git log --decorate=full | grep '^commit.*tag: refs/tags/' | head -1 | sed -r 's|.*refs/tags/|\1|' | sed -r 's|[,)].*||'",
                                                                                returnStdout: true
                                                                        ).trim()
                                                                }
                                                                sh("(echo '## Changes since last version' ; echo ; git shortlog --group=%cd ${lastTag}..HEAD | grep '^      ' | sed 's/^      /* /') | tee ../CHANGELOG.md")
                                                                sh("cat ../CHANGELOG.md | grep -E '^..(fix|feat|BREAKING CHANGE)' > ../SUMMARY.md")
                                                        }
                                                }
                                        }
                                }
                                stage('Prepare manifest') {
                                        steps {
                                                script {
                                                        def factory = "firmware.factory.bin"
                                                        def ota = "firmware.ota.bin"
                                                        def yaml = "digital-audio-bridge.yaml"
                                                        def manifest_template = "src/Code/manifest-template.json"

                                                        def project = sh(
                                                                script: "yq .esphome.project.name ${yaml}",
                                                                returnStdout: true
                                                        ).trim()
                                                        def version = sh(
                                                                script: "yq .esphome.project.version ${yaml}",
                                                                returnStdout: true
                                                        ).trim()
                                                        def min_esphome = sh(
                                                                script: "yq .esphome.min_version ${yaml}",
                                                                returnStdout: true
                                                        ).trim()

                                                        def ota_md5 = sh(
                                                                script: "md5sum ${ota}",
                                                                returnStdout: true
                                                        ).split(" ")[0]
                                                        def ota_sha = sh(
                                                                script: "sha256sum ${ota}",
                                                                returnStdout: true
                                                        ).split(" ")[0]
                                                        def ota_path = "${version}/firmware.ota.bin"
                                                        def factory_md5 = sh(
                                                                script: "md5sum ${factory}",
                                                                returnStdout: true
                                                        ).split(" ")[0]
                                                        def factory_sha = sh(
                                                                script: "sha256sum ${factory}",
                                                                returnStdout: true
                                                        ).split(" ")[0]
                                                        def factory_path = "${version}/firmware.factory.bin"

                                                        sh("""
                                                        cat ${manifest_template} | \
                                                          jq -r --arg project ${shellQuote(project)} '.name |= \$project' | \
                                                          jq -r --arg version ${shellQuote(version)} '.version |= \$version' | \
                                                          jq -r --arg ota_md5 ${shellQuote(ota_md5)} '.builds[0].ota.md5 |= \$ota_md5' | \
                                                          jq -r --arg ota_sha ${shellQuote(ota_sha)} '.builds[0].ota.sha256 |= \$ota_sha' | \
                                                          jq -r --arg ota_path ${shellQuote(ota_path)} '.builds[0].ota.path |= \$ota_path' | \
                                                          jq -r --arg factory_md5 ${shellQuote(factory_md5)} '.builds[0].parts[0].md5 |= \$factory_md5' | \
                                                          jq -r --arg factory_sha ${shellQuote(factory_sha)} '.builds[0].parts[0].sha256 |= \$factory_sha' | \
                                                          jq -r --arg factory_path ${shellQuote(factory_path)} '.builds[0].parts[0].path |= \$factory_path' | \
                                                          jq -r --rawfile changelog SUMMARY.md '.builds[].ota.summary = \$changelog' - | \
                                                          tee manifest.json
                                                        """)
                                                }
                                        }
                                }
                                stage('Publish') {
                                        when { buildingTag() }
                                        steps {
                                                script {
                                                        def yaml = "digital-audio-bridge.yaml"
                                                        def version = sh(
                                                                script: "yq .esphome.project.version ${yaml}",
                                                                returnStdout: true
                                                        ).trim()
                                                        dir('src') {
                                                               def tagName = sh(
                                                                       script: 'git describe --tags --exact-match 2>/dev/null || true',
                                                                       returnStdout: true
                                                               ).trim()
                                                               if (tagName != "v" + version) {
                                                                       error "Tag ${tagName} not the same as version v${version}. Aborting."
                                                              }
                                                        }
                                                        def title = "Version ${version}"
                                                        withCredentials([
                                                                usernamePassword(credentialsId: 'plone-auth', usernameVariable: 'PLONE_USERNAME', passwordVariable: 'PLONE_PASSWORD'),
                                                                string(credentialsId: 'plone-base-url', variable: 'PLONE_BASE_URL'),
                                                        ]) {
                                                                def factory = "firmware.factory.bin"
                                                                def ota = "firmware.ota.bin"

                                                                sh(
                                                                        script: """
                                                                        data=\$(
                                                                        echo '{"@type": "Document", "title": "Version XXX", "id": "XXX", "text": {"content-type": "text/x-web-markdown", "data": "Here goes the changelog"}}' | \
                                                                          jq -r --arg version ${shellQuote(version)} '.id |= \$version' | \
                                                                          jq -r --rawfile changelog CHANGELOG.md '.text.data |= \$changelog' | \
                                                                          jq -r --arg title ${shellQuote(title)} '.title |= \$title'
                                                                        )
                                                                        curl \
                                                                        -XPOST \
                                                                        --fail-with-body \
                                                                        -H "Content-Type: application/json" \
                                                                        -H "Accept: application/json" \
                                                                        --data-raw "\$data" \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL/esphome/digital-audio-bridge"
                                                                        """,
                                                                        label: "Create release document"
                                                                )
                                                                sh(
                                                                        script: """
                                                                        curl \
                                                                        -XPUT \
                                                                        -H "Content-Type: application/octet-stream" \
                                                                        --fail-with-body \
                                                                        --data-binary @${shellQuote(factory)} \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL"/esphome/digital-audio-bridge/${shellQuote(version)}/${shellQuote(factory)}
                                                                        """,
                                                                        label: "Upload factory image"
                                                                )
                                                                sh(
                                                                        script: 
                                                                        """
                                                                        curl \
                                                                        -XPATCH \
                                                                        --fail-with-body \
                                                                        -H "Accept: application/json" \
                                                                        --data-raw '{"title": "Factory firmware", "description": "Flash this image via serial.  Do not flash over the air."}' \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL"/esphome/digital-audio-bridge/${shellQuote(version)}/${shellQuote(factory)}
                                                                        """,
                                                                        label: "Describe factory image"
                                                                )
                                                                sh(
                                                                        script: """
                                                                        curl \
                                                                        -XPUT \
                                                                        -H "Content-Type: application/octet-stream" \
                                                                        --fail-with-body \
                                                                        --data-binary @${shellQuote(ota)} \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL"/esphome/digital-audio-bridge/${shellQuote(version)}/${shellQuote(ota)}
                                                                        """,
                                                                        label: "Upload OTA image"
                                                                )
                                                                sh(
                                                                        script: """
                                                                        curl \
                                                                        -XPATCH \
                                                                        --fail-with-body \
                                                                        -H "Accept: application/json" \
                                                                        --data-raw '{"title": "OTA firmware", "description": "Flash this image over the air.  Do not flash via serial."}' \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL"/esphome/digital-audio-bridge/${shellQuote(version)}/${shellQuote(ota)}
                                                                        """,
                                                                        label: "Describe OTA image"
                                                                )
                                                                sh(
                                                                        script: """
                                                                        curl \
                                                                        -XPOST \
                                                                        --fail-with-body \
                                                                        -H "Accept: application/json" \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL"/esphome/digital-audio-bridge/${shellQuote(version)}/@workflow/publish
                                                                        """,
                                                                        label: "Publish release"
                                                                )
                                                                sh(
                                                                        script: """
                                                                        curl \
                                                                        -XPUT \
                                                                        -H "Content-Type: application/json" \
                                                                        --fail-with-body \
                                                                        --data-binary @manifest.json \
                                                                        -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                        "\$PLONE_BASE_URL"/esphome/digital-audio-bridge/manifest.json
                                                                        """,
                                                                        label: "Update manifest"
                                                                )
                                                        }
                                                }
                                        }
                                }
                                stage('Archive') {
                                        steps {
                                                archiveArtifacts artifacts: 'manifest.json,CHANGELOG.md,SUMMARY.md,*.bin', fingerprint: true
                                        }
                                }

                        }
                }
        }
        post {
                always {
                        announceEnd(currentBuild.currentResult)
                }
        }
}
