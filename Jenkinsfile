// https://github.com/Rudd-O/shared-jenkins-libraries
@Library('shared-jenkins-libraries@master') _

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
                                                git checkout 2026.2.1
                                                script/setup
                                                """)
                                        }
                                }
                                stage('Unstash') {
                                        steps {
                                                dir('src') {
                                                        deleteDir()
                                                }
                                                dir('src') {
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
                                                          jq -r '.name |= "${project}"' | \
                                                          jq -r '.version |= "${version}"' | \
                                                          jq -r '.builds[0].ota.md5 |= "${ota_md5}"' | \
                                                          jq -r '.builds[0].ota.sha256 |= "${ota_sha}"' | \
                                                          jq -r '.builds[0].ota.path |= "${ota_path}"' | \
                                                          jq -r '.builds[0].parts[0].md5 |= "${factory_md5}"' | \
                                                          jq -r '.builds[0].parts[0].sha256 |= "${factory_sha}"' | \
                                                          jq -r '.builds[0].parts[0].path |= "${factory_path}"' | \
                                                          tee manifest.json
                                                        """)
                                                }
                                        }
                                }
                                stage('Publish') {
                                        steps {
                                                script {
                                                        withCredentials([
                                                                usernamePassword(credentialsId: 'plone-auth', usernameVariable: 'PLONE_USERNAME', passwordVariable: 'PLONE_PASSWORD'),
                                                                string(credentialsId: 'plone-base-url', variable: 'PLONE_BASE_URL'),
                                                        ]) {
                                                                def factory = "firmware.factory.bin"
                                                                def ota = "firmware.ota.bin"
                                                                def yaml = "digital-audio-bridge.yaml"
                                                                def version = sh(
                                                                        script: "yq .esphome.project.version ${yaml}",
                                                                        returnStdout: true
                                                                ).trim()
                                                                sh("""
                                                                curl \
                                                                  -XPOST \
                                                                  --fail-with-body \
                                                                  -H "Content-Type: application/json" \
                                                                  -H "Accept: application/json" \
                                                                  --data-raw '{"@type": "Folder", "title": "Version ${version}", "id": "${version}"}' \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge"
                                                                """)
                                                                sh("""
                                                                curl \
                                                                  -XPUT \
                                                                  -H "Content-Type: application/octet-stream" \
                                                                  --fail-with-body \
                                                                  --data-binary @${factory} \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge/${version}/firmware.factory.bin"
                                                                """)
                                                                sh("""
                                                                curl \
                                                                  -XPATCH \
                                                                  --fail-with-body \
                                                                  -H "Accept: application/json" \
                                                                  --data-raw '{"title": "Factory firmware", "description": "Flash this image via serial.  Do not flash over the air."}' \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge/${version}/firmware.factory.bin"
                                                                """)
                                                                sh("""
                                                                curl \
                                                                  -XPUT \
                                                                  -H "Content-Type: application/octet-stream" \
                                                                  --fail-with-body \
                                                                  --data-binary @${ota} \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge/${version}/firmware.ota.bin"
                                                                """)
                                                                sh("""
                                                                curl \
                                                                  -XPATCH \
                                                                  --fail-with-body \
                                                                  -H "Accept: application/json" \
                                                                  --data-raw '{"title": "OTA firmware", "description": "Flash this image over the air.  Do not flash via serial."}' \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge/${version}/firmware.ota.bin"
                                                                """)
                                                                sh("""
                                                                curl \
                                                                  -XPOST \
                                                                  --fail-with-body \
                                                                  -H "Accept: application/json" \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge/${version}/@workflow/publish"
                                                                """)
                                                                sh("""
                                                                curl \
                                                                  -XPUT \
                                                                  -H "Content-Type: application/json" \
                                                                  --fail-with-body \
                                                                  --data-binary @manifest.json \
                                                                  -u "\$PLONE_USERNAME:\$PLONE_PASSWORD" \
                                                                  "\$PLONE_BASE_URL/esphome/digital-audio-bridge/manifest.json"
                                                                """)
                                                        }
                                                }
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
