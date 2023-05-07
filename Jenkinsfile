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
                    RELEASE_VERSION=pomVersion
                    RELEASE_MSG="release ${RELEASE_VERSION}"

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
                GITHUB_REPO_GIT_URL = "https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}.git"
                GITHUB_API_RELEASE_URL  = "https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}/releases"
                GITHUB_CREDS    = credentials('github_ikmdev-pat')
            }

            agent {
                docker {
                    image "maven:3.8.7-eclipse-temurin-19-focal"
                    args '-u root:root'
                }
            }

            steps{

                // set no-reply email address
                sh 'git config --global user.email "120604367+pmaheshm@users.noreply.github.com"'
                sh 'git config --global user.name "pmaheshm"'
                sh 'git config --global http.sslVerify false'
                // sh 'git config --global http.version HTTP/1.1'

                // See what remotes are currently present
                sh 'git remote -v'

                // Get the latest tag
                //sh 'git describe --abbrev=0 --tags'

                // Tag the branch
                //sh "git tag -a ${RELEASE_VERSION} -m '${RELEASE_MSG}'"

                withCredentials([gitUsernamePassword(credentialsId: 'gitlab-for-ikmdev-release-token', gitToolName: '')]) {
                    // Tag the origin repo
                    sh "git branch -b"
                    sh "git checkout -b main"
                    sh "git push origin ${RELEASE_VERSION}"
                }

                withCredentials([gitUsernamePassword(credentialsId: 'github_ikmdev-pat', gitToolName: '')]) {
                    // reset the author information on your last commit
                    //sh "git commit -m 'committing code for ${RELEASE_MSG}'"

                    // Push the new tag to downstream remote
                    sh "git push downstream ${RELEASE_VERSION}"

                    // Push the branch to downstream remote
                    sh "git push downstream ${BRANCH}"
                }

                sh """
                    set -x

                    curl -u ${GITHUB_CREDS_USR}:${GITHUB_CREDS_PSW} -X POST \
                    -H 'Accept: application/vnd.github.v3+json' ${GITHUB_API_RELEASE_URL} \
                    -d '{"tag_name":"${RELEASE_VERSION}","target_commitish":"main","name":"${RELEASE_MSG}","draft":false,"prerelease":false,"generate_release_notes":false,"body":"${RELEASE_MSG}"}'

                    echo "pushed to ikm github successfully"
                """
            }
        }
    }
}
