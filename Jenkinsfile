@Library("titan-library") _

pipeline {
    agent any

    environment {
        GITLAB_REPO     = 'https://gitlab.tinkarbuild.com/test-group/test-publish-gitlab-github.git'

        WORKING_DIR     = 'sourceRepo'
        BRANCH          = 'main'
        RELEASE_NOTE    = ''
        VERSION         = ''
        MSG             = ''
        WEBHOOK_URL     = "${GLOBAL_CHATOPS_URL}"
    }

    options {

        // Set this to true if you want to clean workspace during the prep stage
        skipDefaultCheckout(false)

        // Console debug options
        timestamps()
        ansiColor('xterm')

    }

    stages {

        stage('Maven Build') {
            agent {
                docker {
                    image "maven:3.8.7-eclipse-temurin-19-alpine"
                    args '-u root:root'
                }
            }

            steps {
                script{
                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                        mvn clean install -s '${MAVEN_SETTINGS}' \
                            --batch-mode \
                            -e \
                            -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn
                        """
                    }
                }
            }
        }

        stage("Publish to Nexus Repository Manager") {

            agent {
                docker {
                    image "maven:3.8.7-eclipse-temurin-19-focal"
                    args '-u root:root'
                }
            }

            steps {

                script {
                    pomModel = readMavenPom(file: 'pom.xml')
                    pomVersion = pomModel.getVersion()
                    isSnapshot = pomVersion.contains("-SNAPSHOT")
                    repositoryId = 'maven-releases'

                    if (isSnapshot) {
                        repositoryId = 'maven-snapshots'
                    }

                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {

                        sh """
                            mvn deploy \
                            --batch-mode \
                            -e \
                            -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                            -DskipTests \
                            -DskipITs \
                            -Dmaven.main.skip \
                            -Dmaven.test.skip \
                            -Dmaven.javadoc.skip \
                            -s '${MAVEN_SETTINGS}' \
                            -P inject-application-properties \
                            -DrepositoryId='${repositoryId}'
                        """
                    }
                }
            }
        }

        stage("Release to ikm github") {

            when{
                expression{
                    branch == 'main' && !isSnapshot
                }
            }

            environment {
                GITHUB_OWNER    = "ikmdev"
                GITHUB_REPO     = "tinkar-starter-bindings"
                GITHUB_REPO_GIT = "https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}.git"
                GITHUB_RELEASE  = "https://api.github.com/repos/${GITHUB_OWNER}/test-publish-gitlab-github/releases"
                GITHUB_CREDS    = credentials('github_ikmdev-pat')
            }

            agent {
                docker {
                    image "maven:3.8.7-eclipse-temurin-19-focal"
                    args '-u root:root'
                }
            }

            steps{
                sh """
                    set -x
                    curl -u ${GITHUB_CREDS_USR}:${GITHUB_CREDS_PSW} -X POST -H 'Accept: application/vnd.github.v3+json' https://api.github.com/repos/ikmdev/test-publish-gitlab-github/releases -d '{"tag_name":"${VERSION}","target_commitish":"main","name":"${MSG}","draft":false,"prerelease":false,"generate_release_notes":false,"body":"${MSG}"}'
                    echo "pushed to ikm github successfully"
                """
            }
        }
    }
}